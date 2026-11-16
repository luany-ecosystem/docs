# Luany Documentation

The official documentation for the [Luany](https://github.com/luany-ecosystem) PHP framework,
in English and Portuguese. Read it at **[docs.luany.dev](https://docs.luany.dev)**.

| Section | Pages |
| --- | --- |
| **Getting Started** | [Installation](en/installation.md), [Configuration](en/configuration.md), [Directory Structure](en/directory-structure.md), [Request Lifecycle](en/request-lifecycle.md) |
| **The Basics** | [Routing](en/routing.md), [Middleware](en/middleware.md), [Controllers](en/controllers.md), [Requests & Responses](en/requests.md), [Validation](en/validation.md), [CSRF Protection](en/csrf.md), [Session & Flash](en/session.md), [Error Handling](en/errors.md), [Logging](en/logging.md), [Localization](en/localization.md), [Helpers](en/helpers.md) |
| **Views (LTE)** | [Template Syntax](en/lte-syntax.md), [Layouts & Components](en/lte-layouts.md), [Directives Reference](en/lte-directives.md), [Assets & Stacks](en/lte-assets.md), [Custom Directives](en/lte-custom.md), [Template Errors](en/lte-errors.md) |
| **Database** | [Getting Started](en/database.md), [Query Builder](en/query-builder.md), [Models](en/models.md), [Relationships](en/relationships.md), [Migrations & Schema](en/migrations.md), [Seeding](en/seeding.md), [Pagination](en/pagination.md) |
| **CLI** | [Commands](en/cli.md), [make:feature](en/make-feature.md), [Dev Server & Live Reload](en/dev-server.md), [view:check](en/view-check.md) |
| **Advanced** | [Service Container](en/container.md), [Service Providers](en/providers.md), [PSR Interoperability](en/psr.md), [Security](en/security.md), [Deployment](en/deployment.md), [VS Code Extension](en/vscode.md) |

Portuguese: same file names under [`pt/`](pt/).

## Layout

```
manifest.json      sections, page order and titles (en / pt)
en/{slug}.md       English pages
pt/{slug}.md       Portuguese pages
STYLE.md           how pages are written
```

The site ([docs.luany.dev](https://github.com/luany-ecosystem/docs.luany.dev)) renders this
repository as is; a push to `main` is live on the next pull.

## Contributing

Corrections are welcome — every page has an **Edit this page** link. Before opening a pull request:

1. Follow [STYLE.md](STYLE.md) and check every example against the current releases.
2. Change both languages when you change behaviour descriptions (or note the missing translation in the PR).
3. Check links and anchors from a docs.luany.dev checkout: `php bin/check-content.php ../docs`.

## License

MIT
