# ABP Community Articles

A collection of articles written for the [ABP Community](https://abp.io/community) —
tutorials, how-tos, and deep dives on the [ABP Framework](https://abp.io) and the
.NET ecosystem.

## Articles

| Date | Article | Topics |
| ---- | ------- | ------ |
| 2026-06-07 | [Enabling MCP Authentication in ABP Framework with OpenIddict](./2026-06-07-Enabling-MCP-Authentication-in-ABP-with-OpenIddict/POST.md) | OpenIddict · MCP · OAuth 2.1 · PKCE · RFC 8707 · ASP.NET Core |

## Repository layout

Each article lives in its own dated folder, following the ABP community convention:

```
YYYY-MM-DD-Title-Slug/
├── POST.md        # the article (Markdown, no frontmatter — title is the first H1)
└── cover.png      # cover image (PNG/JPG, < 1MB) — optional but recommended
```

- **`POST.md`** starts directly with a single `# Title` heading; author/byline and
  publish metadata are handled by the ABP Community platform, not in the file.
- Images are referenced with relative paths (e.g. `![alt](./screenshot.png)`) and
  kept under 1 MB.

## Contributing / adding an article

1. Create a new `YYYY-MM-DD-Title-Slug/` folder.
2. Add `POST.md` (and a `cover.png` if you have one).
3. Add a row to the **Articles** table above.
4. Commit and push.

## License

See [LICENSE](./LICENSE).
