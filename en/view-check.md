# view:check

`luany view:check` validates every LTE view of a project without rendering anything, and exits with a non-zero code when a view is broken. Use it locally, in your editor and in CI.

## Usage

```bash
luany view:check
```

```bash
  →  Checking 9 view(s) in views/

  ✗  views/pages/products/index.lte:42:21
     Unclosed @foreach: missing @endforeach.

  ✗  1 of 9 view(s) have errors.
```

When everything is valid:

```bash
  →  Checking 9 view(s) in views/

  ✓  9 view(s) OK.
```

The command requires `luany/lte` 1.2 or newer in the project (`composer update luany/lte` otherwise) and luany/cli 1.4 or newer.

## What is checked

Each `.lte` file under the views directory (recursively, in alphabetical order) is parsed and validated — nothing is executed and no compiled file is written:

- every block is closed by the right directive; no stray or crossed closers (`@if` … `@endforeach`);
- `@elseif` / `@else` only inside `@if`-style blocks, a bare `@empty` only once inside `@forelse`;
- directives that need arguments have them (`@include`, `@section`, …);
- the PHP inside `{{ }}`, `{!! !!}`, `@php` and directive arguments is valid syntax.

Errors are reported as `file:line:column` with the innermost message. Runtime problems — an undefined variable, a missing included view — are not detected, because nothing is rendered. See [Template Errors](lte-errors.md) for the full list of compile-time messages.

## Options

| Option | Default | Description |
|---|---|---|
| `--path=<dir>` | `views` | Views directory, relative to the project root. `..` is not allowed. |
| `--format=text\|github` | `text` | `github` prints GitHub Actions annotations instead of the human-readable report |
| `--no-app` | off | Do not boot the application; use a plain LTE engine |

Unknown options are rejected with a usage message.

```bash
luany view:check --no-app
luany view:check --path=resources/views
luany view:check --format=github
```

## Application boot

By default `view:check` boots your application — it requires `bootstrap/app.php`, boots the HTTP Kernel and uses the bound `view` engine. That way **custom directives registered by your service providers** are known and validated like built-in ones. All output produced while booting is discarded.

If booting fails (for example `.env` is missing and `Env::required()` throws), the command falls back to a plain engine and tells you:

```bash
  ⚠  Could not boot the application (Required environment variables are not set: APP_ENV, APP_URL).
     Checking without directives registered by your providers.
```

In that mode, and with `--no-app`, any `@name` that is not a built-in directive is treated as plain text, so a view using custom directives is still checked for structure and PHP syntax.

With the application booted, files under `--path` are parsed by path with the application's compiler, so custom directives are still known.

> **Note** Before luany/cli 1.6, files under a custom `--path` were reported as "not found" when the application booted. On older versions, combine `--path` with `--no-app`.

`--no-app` is faster and has no side effects; it is what the VS Code extension uses by default.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | All views are valid |
| `1` | At least one view has errors, the views directory does not exist, `luany/lte` is older than 1.2, an option is invalid, or the command was run outside a Luany project |

## GitHub annotations

With `--format=github`, the summary lines are suppressed and each error becomes a workflow command:

```bash
::error file=views/pages/products/index.lte,line=42,col=21::Unclosed @foreach: missing @endforeach.
```

GitHub Actions turns these into annotations on the pull request's changed lines. The output is also stable for other tools — the [VS Code extension](vscode.md) parses it to underline errors in the editor.

## CI example

The skeleton does not depend on `luany/cli`, so install it in the job. This GitHub Actions workflow checks the views on every push and pull request:

```yaml
name: Views

on: [push, pull_request]

jobs:
  view-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: shivammathur/setup-php@v2
        with:
          php-version: "8.2"
          extensions: pdo, pdo_sqlite, mbstring

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Install the Luany CLI
        run: composer global require luany/cli --no-interaction

      - name: Prepare .env
        run: cp .env.example .env

      - name: Check LTE views
        run: $(composer global config bin-dir --absolute)/luany view:check --format=github
```

The `.env` step lets the application boot so provider-registered directives are validated. If your providers need services that are not available in CI (a database server, API keys), drop that step and add `--no-app`.

Alternatively, add the CLI as a development dependency and run `vendor/bin/luany view:check --format=github`:

```bash
composer require --dev luany/cli
```

## Programmatic use

The command is built on the engine's `parseView()` method, which you can call from your own tooling:

```php
use Luany\Lte\Engine;
use Luany\Lte\TemplateSyntaxException;

$engine = new Engine(__DIR__ . '/views', sys_get_temp_dir() . '/lte-check');

try {
    $engine->parseView('pages.home');
} catch (TemplateSyntaxException $e) {
    echo "{$e->view}:{$e->templateLine}:{$e->templateColumn} {$e->getMessage()}\n";
}
```
