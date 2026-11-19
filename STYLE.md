# Writing guide

Pages live in `en/{slug}.md` and `pt/{slug}.md`; navigation and titles come from `manifest.json`.

- Start with `# Title` (same as the manifest), then one or two sentences on what the page covers.
- Use `##` / `###` for sections; the site builds the "On this page" outline from them.
- Every code sample must work with the current releases: core 1.4, framework 1.5, lte 1.3,
  database 1.8, cli 1.7, skeleton 1.5, VS Code extension 1.2. Check signatures in the package source — never invent an API.
- Fence code with a language: ```php, ```lte (LTE templates), ```bash, ```env, ```json.
- Link other pages with relative links: `[Routing](routing.md)`, `[Models](models.md#casts)`.
- Callouts: a blockquote starting with **Note**, **Tip** or **Warning**:
  `> **Warning** Raw methods are not escaped.`
- Prefer short paragraphs, tables for option lists, and one runnable example per concept.
- English is the reference; Portuguese pages translate it faithfully (pt-PT / pt-AO).
