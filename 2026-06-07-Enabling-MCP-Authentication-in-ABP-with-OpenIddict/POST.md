---
title: Enabling MCP Authentication in ABP Framework with OpenIddict
tags:
  - mcp
  - oauth
---
# Enabling MCP Authentication in ABP Framework with OpenIddict

Every ABP Framework application already ships with a fully configured OAuth 2.1 / OpenID Connect **Authorization Server** — the bundled [OpenIddict module](https://abp.io/docs/latest/modules/openiddict). We usually think of it as the thing that logs users into the web UI and issues tokens to the Swagger client. But it is a real, spec-compliant authorization server, and that turns out to be exactly what the **Model Context Protocol (MCP)** needs.

The newest wave of AI tooling — Claude Desktop, the VS Code agent, the MCP Inspector — talks to your backend through MCP servers, and the current MCP authorization spec requires those servers to be protected by OAuth 2.1 with PKCE and Resource Indicators. Rather than standing up a separate identity provider, you can reuse the one your ABP app already runs. In this article I'll show you how, end to end, using nothing but the OpenIddict data-seed contributor and a single `PreConfigure` block in the host module. No custom middleware, no hand-rolled token endpoints.

We'll use the classic `Acme.BookStore` sample as our ABP application. By the end you'll have:

- an OAuth **scope** the MCP server exposes (`BookStoreMcp`);
- a **public client** for interactive tools (authorization code + PKCE);
- a **confidential client** for machine-to-machine testing;
- **RFC 8707 Resource Indicators** wired up so the issued tokens carry the right `aud` for your MCP server.

Every line of code you need is in this article — including a minimal ASP.NET Core MCP server in the appendix, so you have a resource server to point the client at.

> Tested with **ABP Framework 10.x** on **.NET 10**.

## A 2-minute primer: why MCP needs OAuth

The **Model Context Protocol (MCP)** is the open protocol that lets AI clients (Claude, VS Code, IDE agents) call your "tools" and read your "resources" over a standard interface. An **MCP server** exposes those tools; an **MCP client** consumes them.

As of the current MCP authorization specification, an MCP server is treated as an OAuth 2.0 **protected resource**, and clients must obtain a token before calling it. The spec leans on a stack of well-established RFCs — and the good news is that ABP's OpenIddict module already implements all of them:

| Concept | What it does in MCP | Who provides it |
| --- | --- | --- |
| **OAuth 2.1 + PKCE** | Interactive clients get a token via authorization code + PKCE (no client secret) | ABP / OpenIddict |
| **Authorization Server Metadata** (RFC 8414) | Client discovers `/connect/token`, `/connect/authorize`, JWKS, etc. | OpenIddict exposes `/.well-known/openid-configuration` automatically |
| **Protected Resource Metadata** (RFC 9728) | The MCP server tells the client *which* authorization server to use | The MCP server (see appendix) |
| **Resource Indicators** (RFC 8707) | The client says "I want a token *for this MCP server*"; the server stamps it as the token `aud` | ABP / OpenIddict |

The three roles involved:

```
┌────────────────┐   1. OAuth 2.1 + PKCE   ┌─────────────────────────┐
│  MCP Client    │ ──────────────────────► │  ABP App                │
│  (Claude /     │                         │  = Authorization Server │
│   VS Code /    │ ◄────────────────────── │  (OpenIddict)           │
│   Inspector)   │   2. access_token (JWT) │  /connect/authorize     │
└──────┬─────────┘     aud = MCP server    │  /connect/token         │
       │                                   └─────────────────────────┘
       │ 3. call tools (Authorization: Bearer <jwt>)
       ▼
┌──────────────────────┐
│  Your MCP Server     │  = Resource Server
│  validates aud == me │  (appendix)
└──────────────────────┘
```

Your ABP app is **only** the Authorization Server here. It mints the token; the MCP server (a separate process) validates it. Keep that separation in mind — it's the reason we configure *two* "resources" lists below.

## Prerequisites

- **.NET 10 SDK**
- An **ABP Framework 10.x** application created from the standard startup template (the layered solution with `*.HttpApi.Host` and `*.DbMigrator`). The template already includes and configures the OpenIddict module — we only customize it. If you don't have one yet:
  ```bash
  abp new Acme.BookStore -t app
  ```
- **Node.js**, only to run the MCP Inspector (`npx @modelcontextprotocol/inspector`).

## Step 1 — Define the scope the MCP server exposes

Open the seed contributor the template generated for you — for `Acme.BookStore` it's `Acme.BookStore.OpenIddict.BookStoreOpenIddictDataSeedContributor` in the **`*.Domain`** project. We'll add a method that seeds an MCP scope and the MCP clients, and call it from `SeedAsync`.

Add these `using` directives at the top of the file:

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.Extensions.Configuration;
using OpenIddict.Abstractions;
using Volo.Abp.DependencyInjection;
using Volo.Abp.OpenIddict.Applications;
using Volo.Abp.OpenIddict.Scopes;
using Volo.Abp.Uow;
using static OpenIddict.Abstractions.OpenIddictConstants;
```

The scope's **`Resources`** are the canonical URL(s) of your MCP server. OpenIddict stamps the granted resource as the token's `aud`, so this is how the MCP server later recognizes "this token is for me." Capture the base-class managers in the constructor and seed the scope:

```csharp
public class BookStoreOpenIddictDataSeedContributor
    : OpenIddictDataSeedContributorBase, IDataSeedContributor, ITransientDependency
{
    private readonly IOpenIddictScopeManager _scopeManager;
    private readonly IAbpApplicationManager _applicationManager;

    public BookStoreOpenIddictDataSeedContributor(
        IConfiguration configuration,
        IOpenIddictApplicationRepository applicationRepository,
        IAbpApplicationManager applicationManager,
        IOpenIddictScopeRepository scopeRepository,
        IOpenIddictScopeManager scopeManager)
        : base(configuration, applicationRepository, applicationManager, scopeRepository, scopeManager)
    {
        _scopeManager = scopeManager;
        _applicationManager = applicationManager;
    }

    [UnitOfWork]
    public override async Task SeedAsync(DataSeedContext context)
    {
        await CreateScopesAsync();          // generated by the template
        await CreateApplicationsAsync();    // generated by the template
        await CreateMcpResourcesAsync();    // our new method
    }

    private async Task CreateMcpResourcesAsync()
    {
        // The MCP server's own canonical URL(s). Register BOTH the trailing-slash
        // and no-slash forms — the client echoes the value verbatim and different
        // SDKs normalize it differently, so an ordinal match must succeed either way.
        var mcpScope = new OpenIddictScopeDescriptor
        {
            Name        = "BookStoreMcp",
            DisplayName = "BookStore MCP API access",
            Resources   =
            {
                "https://localhost:7071",
                "https://localhost:7071/",        // dev MCP server
                "https://mcp.example.com",
                "https://mcp.example.com/"        // production MCP server
            }
        };

        // Create OR update so re-running the migrator re-applies Resources changes.
        var existingScope = await _scopeManager.FindByNameAsync("BookStoreMcp");
        if (existingScope is null)
        {
            await _scopeManager.CreateAsync(mcpScope);
        }
        else
        {
            await _scopeManager.UpdateAsync(existingScope, mcpScope);
        }

        await CreateMcpClientsAsync();
    }

    // CreateMcpClientsAsync() is shown in Step 2.
}
```

> **Tip — create *or* update.** The DbMigrator runs against an existing database. If you only `CreateAsync` when the scope is missing, later edits to `Resources` are silently skipped and you end up editing the DB by hand. The `FindByNameAsync` → `UpdateAsync` pattern makes the seed the single source of truth.

## Step 2 — Register the MCP clients

We seed two clients: one for **interactive** tools and one for **machine-to-machine** testing. Add this method to the same contributor class:

```csharp
private async Task CreateMcpClientsAsync()
{
    // 1) Interactive client (Claude Desktop / VS Code / MCP Inspector).
    //    Public client (no secret); PKCE is mandatory.
    var interactiveClient = new OpenIddictApplicationDescriptor
    {
        ClientId    = "BookStore_McpClient",
        ClientType  = ClientTypes.Public,        // PKCE, no secret
        ConsentType = ConsentTypes.Explicit,
        DisplayName = "BookStore MCP Client",

        // Loopback callbacks the client listens on. The MCP Inspector runs at
        // :6274 and builds its callback as `window.location.origin + "/oauth/callback"`,
        // with a "/debug" variant for its Guided OAuth Flow. Register both the
        // localhost and 127.0.0.1 hosts so it works either way.
        RedirectUris =
        {
            new Uri("http://localhost:6274/oauth/callback"),
            new Uri("http://localhost:6274/oauth/callback/debug"),
            new Uri("http://127.0.0.1:6274/oauth/callback"),
            new Uri("http://127.0.0.1:6274/oauth/callback/debug")
        },

        Permissions =
        {
            Permissions.Endpoints.Authorization,
            Permissions.Endpoints.Token,

            Permissions.GrantTypes.AuthorizationCode,
            Permissions.GrantTypes.RefreshToken,        // enables offline_access

            Permissions.ResponseTypes.Code,

            Permissions.Scopes.Profile,
            Permissions.Scopes.Email,
            Permissions.Prefixes.Scope + "BookStoreMcp" // our scope
        },

        Requirements =
        {
            Requirements.Features.ProofKeyForCodeExchange   // force PKCE
        }
    };

    var existingInteractive = await _applicationManager.FindByClientIdAsync("BookStore_McpClient");
    if (existingInteractive is null)
        await _applicationManager.CreateAsync(interactiveClient);
    else
        await _applicationManager.UpdateAsync(existingInteractive, interactiveClient);

    // 2) Confidential client for quick machine-to-machine testing (see Step 5).
    if (await _applicationManager.FindByClientIdAsync("BookStore_McpTest") is null)
    {
        await _applicationManager.CreateAsync(new OpenIddictApplicationDescriptor
        {
            ClientId     = "BookStore_McpTest",
            ClientSecret = "<your-dev-secret>",   // DEV ONLY — never ship this
            ClientType   = ClientTypes.Confidential,
            DisplayName  = "BookStore MCP Test (m2m)",
            Permissions  =
            {
                Permissions.Endpoints.Token,
                Permissions.GrantTypes.ClientCredentials,
                Permissions.Prefixes.Scope + "BookStoreMcp"
            }
        });
    }
}
```

A few notes:

- **`ProofKeyForCodeExchange`** makes PKCE mandatory — required by OAuth 2.1 and by the MCP spec for public clients.
- The **`RefreshToken`** grant enables `offline_access` so the tool can keep a session without re-prompting consent every time.
- The **confidential** `BookStore_McpTest` client is only for the `curl` smoke test in Step 5. Keep its secret out of source control in a real project.

## Step 3 — Register the MCP server URLs as accepted resources (RFC 8707)

This is the step most people miss, so it deserves a clear explanation. There are **two different lists** called "Resources," and they do different jobs:

| List | Where you set it | What it validates |
| --- | --- | --- |
| **Server-options resources** | `OpenIddictServerBuilder.RegisterResources(...)` in the **host module** | The `resource` **request parameter** the client sends on authorize/token |
| **Scope resources** | The `OpenIddictScopeDescriptor.Resources` we seeded in Step 1 | Nothing on the request — they only populate the token **`aud`** |

When an MCP client requests a token, it sends `resource=https://mcp.example.com/` (the MCP server URL it discovered from Protected Resource Metadata). OpenIddict **validates that parameter against the server-options list**. If the URL isn't registered there, you get `ID2190 invalid_target` and the whole flow fails.

Add this to your host module's `PreConfigureServices` (e.g. `BookStoreHttpApiHostModule` in the `*.HttpApi.Host` project):

```csharp
using Microsoft.Extensions.Configuration;
using OpenIddict.Server;   // OpenIddictServerBuilder
// ...

public override void PreConfigureServices(ServiceConfigurationContext context)
{
    var configuration = context.Services.GetConfiguration();

    PreConfigure<OpenIddictServerBuilder>(serverBuilder =>
    {
        // Accepted resource URLs come from configuration, so they can differ per
        // environment (dev -> appsettings.json, prod -> appsettings.Production.json).
        var resources = configuration.GetSection("OpenIddict:Resources").Get<string[]>();
        if (resources is { Length: > 0 })
        {
            serverBuilder.RegisterResources(resources);
        }

        // RegisterResources fixes ID2190 (invalid_target). ID2192 ("client not
        // allowed to use the resource") is a SEPARATE per-client resource-permission
        // check. Access is already gated by the BookStoreMcp scope permission on
        // each client, so we don't also maintain per-client resource permissions.
        serverBuilder.IgnoreResourcePermissions();
    });
}
```

> **Hardening option.** If you'd rather enforce per-client resource permissions, drop `IgnoreResourcePermissions()` and instead grant each client the explicit permission `Permissions.Prefixes.Resource + "https://mcp.example.com/"` in its descriptor.

## Step 4 — Configure the resource URLs per environment

`OpenIddict:Resources` is read from configuration so you don't hard-code hostnames.

```jsonc
// appsettings.json (Development)
"OpenIddict": {
  "Resources": [ "https://localhost:7071/" ]
}
```

```jsonc
// appsettings.Production.json
"OpenIddict": {
  "Resources": [ "https://mcp.example.com/" ]
}
```

> **Keep these in sync** with the **scope** `Resources` from Step 1. The server-options list (this config) decides which `resource` values are *accepted*; the scope list decides what ends up in the token `aud`. When you add a new MCP server host, update **both**.

## Step 5 — Run the migrator and verify

Apply the seed by running the DbMigrator:

```bash
cd src/Acme.BookStore.DbMigrator
dotnet run
```

> **If your migrator's console stays silent:** the ABP template logs through Serilog's `WriteTo.Async(...)`, which buffers on a background thread. Add a flush at the very end of `Program.Main` so your seed logs aren't dropped when the process exits:
> ```csharp
> await CreateHostBuilder(args).RunConsoleAsync();
> Log.CloseAndFlush();   // flush buffered async-sink lines before exit
> ```

### 5a — Machine-to-machine smoke test (curl)

The fastest way to confirm the scope, client, and resource registration all line up is a client-credentials call with the confidential test client:

```bash
curl -k -X POST https://localhost:44300/connect/token \
  -d grant_type=client_credentials \
  -d client_id=BookStore_McpTest \
  -d client_secret='<your-dev-secret>' \
  -d scope=BookStoreMcp \
  -d resource=https://mcp.example.com/
```

A `200` with an `access_token` means everything is wired correctly. Decode the JWT (for example at [jwt.io](https://jwt.io)) and confirm the `aud` claim equals your MCP server URL and `scope` contains `BookStoreMcp`.

### 5b — Interactive flow (MCP Inspector)

Run the Inspector and point it at your MCP server:

```bash
npx @modelcontextprotocol/inspector
```

When you connect to a protected MCP server, the Inspector discovers your ABP app from the server's Protected Resource Metadata, opens the ABP login page, you sign in and consent, and it completes the `authorization_code` + PKCE exchange against `https://localhost:44300/connect/token`. The token it receives is scoped to `BookStoreMcp` with `aud` = your MCP server URL.

## Conclusion

Your ABP application was already a capable OAuth 2.1 authorization server, and MCP authentication is just a matter of telling it about **one scope**, **two clients**, and the **resource URLs** your MCP server answers to. No custom auth code — the OpenIddict module does the heavy lifting, and the seed contributor keeps the configuration reproducible across environments.

From here you can grant the MCP scope to whichever interactive tools your team uses, add more resource URLs as you deploy MCP servers, and rely on standard ABP permission checks inside the tools your MCP server exposes.

If you try this in your own solution, I'd love to hear how it goes — drop a comment with your setup or any rough edges you hit.

## Appendix — A minimal ASP.NET Core MCP server (the resource server)

The article above configured the **authorization server** (your ABP app). To have something to actually sign in to, here's a minimal **resource server**: an ASP.NET Core MCP server that validates the ABP-issued JWT and advertises its authorization server via Protected Resource Metadata. It's a single `Program.cs`.

> The official C# MCP SDK evolves quickly — treat the SDK calls (`AddMcpServer`, `WithHttpTransport`, `MapMcp`, the `[McpServerTool]` attributes) as a guide and check the latest [`modelcontextprotocol/csharp-sdk`](https://github.com/modelcontextprotocol/csharp-sdk) docs for the current API surface. The auth/PRM parts are plain ASP.NET Core and are stable.

### A.1 — Create the project and add packages

```bash
dotnet new web -n Acme.BookStore.McpServer
cd Acme.BookStore.McpServer
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package ModelContextProtocol.AspNetCore   # official MCP server SDK
```

### A.2 — The complete Program.cs

```csharp
using System.ComponentModel;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using ModelContextProtocol.Server;   // [McpServerTool], [McpServerToolType]

var builder = WebApplication.CreateBuilder(args);

const string McpServerUrl = "https://localhost:7071/";   // THIS server's canonical URL
const string AuthServer   = "https://localhost:44300";   // your ABP app (authorization server)

// --- Validate the ABP-issued token (this is a pure resource server) ---
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = AuthServer;     // discovers signing keys via /.well-known/openid-configuration
        options.Audience  = McpServerUrl;   // must equal the token `aud`
        options.TokenValidationParameters.ValidateAudience = true;

        // Per the MCP spec, point unauthenticated callers at this resource server's
        // metadata (RFC 9728) so they can discover the authorization server.
        options.Events = new JwtBearerEvents
        {
            OnChallenge = context =>
            {
                context.HandleResponse();
                context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                context.Response.Headers.WWWAuthenticate =
                    $"Bearer resource_metadata=\"{McpServerUrl}.well-known/oauth-protected-resource\"";
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

// --- Register the MCP server, HTTP transport, and your tools ---
builder.Services
    .AddMcpServer()
    .WithHttpTransport()
    .WithTools<BookStoreTools>();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// --- RFC 9728: Protected Resource Metadata — which authorization server to use ---
app.MapGet("/.well-known/oauth-protected-resource", () => Results.Json(new
{
    resource = McpServerUrl,
    authorization_servers = new[] { AuthServer }
}));

// Require a valid ABP-issued token on the MCP endpoints.
app.MapMcp().RequireAuthorization();

app.Run();

[McpServerToolType]
public class BookStoreTools
{
    [McpServerTool, Description("Returns a friendly greeting.")]
    public string Hello(string name) => $"Hello, {name}, from the BookStore MCP server!";
}
```

What the three pieces do:

- **JWT bearer** — `Authority` is your ABP app; the handler fetches its signing keys automatically and rejects any token whose `aud` isn't this server's URL.
- **`OnChallenge`** — emits `WWW-Authenticate: Bearer resource_metadata="…"` on a 401 so an MCP client can find the metadata document below.
- **Protected Resource Metadata** — the `/.well-known/oauth-protected-resource` document tells the client which authorization server (your ABP app) to use.

### A.3 — Run it end to end

1. Run your ABP app (`Acme.BookStore.HttpApi.Host`) on `https://localhost:44300`.
2. Run the MCP server on `https://localhost:7071`.
3. Run the MCP Inspector (`npx @modelcontextprotocol/inspector`) and connect to `https://localhost:7071`.
4. The Inspector reads the PRM, redirects you to the ABP login, you consent, and it calls your `Hello` tool with a valid token.

Because the token's `aud` is `https://localhost:7071/` (set by the scope's `Resources` in Step 1) and the MCP server validates that exact audience, only tokens minted *for this server* are accepted — which is the whole point of RFC 8707 Resource Indicators.

## References

- [ABP OpenIddict Module](https://abp.io/docs/latest/modules/openiddict)
- [Model Context Protocol — Authorization specification](https://modelcontextprotocol.io/specification)
- [MCP C# SDK (`modelcontextprotocol/csharp-sdk`)](https://github.com/modelcontextprotocol/csharp-sdk)
- [RFC 8707 — Resource Indicators for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc8707)
- [RFC 9728 — OAuth 2.0 Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728)
- [RFC 8414 — OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414)
- [OpenID Connect Core — Client Authentication](https://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication)
