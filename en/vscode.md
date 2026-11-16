# VS Code Extension

The official **Luany LTE** extension for Visual Studio Code adds syntax highlighting, snippets, a file icon and live error diagnostics for `.lte` templates. The file icon and the diagnostics were added in version 1.2. This page covers installation, features and settings.

## Installation

### From the Marketplace

1. Open the Extensions view (`Ctrl+Shift+X`, or `Cmd+Shift+X` on macOS).
2. Search for **Luany LTE** (publisher `luany`).
3. Click **Install**.

Or from the terminal:

```bash
code --install-extension luany.luany-lte
```

### From a VSIX file

Download the `.vsix` package from the extension's GitHub releases, then:

```bash
code --install-extension luany-lte-1.2.0.vsix
```

or use **Extensions → … → Install from VSIX…** in the editor. The extension requires VS Code 1.80 or newer and activates for `.lte` files and for workspaces that contain them.

## Syntax highlighting

`.lte` files get their own language mode (**Luany LTE**) with:

- every LTE directive highlighted, with PHP arguments at any parenthesis depth;
- embedded PHP inside `{{ }}`, `{!! !!}`, `@php` blocks and directive arguments;
- `{{ }}` and `{!! !!}` recognised in text, HTML attribute values, `<script>` and `<style>`;
- embedded CSS inside `@style` blocks and JavaScript inside `@script` blocks;
- the escapes `@@` and `@{{ }}` shown as literals; e-mail addresses and other `word@word` text never mistaken for directives.

The extension also configures auto-indentation, auto-closing pairs for LTE constructs and Emmet HTML abbreviations in `.lte` files. Your own editor settings, such as word wrap and format-on-type, are left alone.

## Snippets

Type a prefix and press `Tab`. Prefixes in parentheses are aliases.

| Prefix | Inserts |
|---|---|
| `layout` | Official LTE view structure |
| `component` | Self-contained component with collocated CSS and JS |
| `component-use` (`@component`) | Component with a named slot |
| `@slot` (`slot`) | Named slot inside a component |
| `if` | `@if` / `@endif` |
| `ifelse` | `@if` / `@else` / `@endif` |
| `unless` (`@unless`) | `@unless` / `@endunless` |
| `isset` (`@isset`) | `@isset` / `@endisset` |
| `ifempty` (`@ifempty`) | `@ifempty` / `@endifempty` |
| `foreach` | `@foreach` / `@endforeach` |
| `forelse` | `@forelse` / `@empty` / `@endforelse` with an empty state |
| `for` (`@for`) | `@for` / `@endfor` |
| `while` (`@while`) | `@while` / `@endwhile` |
| `section` | `@section` / `@endsection` |
| `style` | `@style` block (collected by the asset stack) |
| `script` | `@script(defer)` block (collected by the asset stack) |
| `push` | `@push` / `@endpush` |
| `include` | `@include` a partial |
| `include-data` | `@include` a partial with data |
| `php` (`@php`) | `@php` / `@endphp` |
| `@json` (`json`) | Safe JSON output for inline JavaScript |
| `@class` (`class`) | Conditional CSS classes |
| `@method` (`method`) | HTTP method spoofing field |
| `form` | `POST` form with `@csrf` |
| `form-method` | Form with `@csrf` and `@method` |
| `csrf` | CSRF hidden field |
| `auth` | `@auth` / `@endauth` |
| `guest` | `@guest` / `@endguest` |
| `dump` (`@dump`) | `@dump($var)` — no-op when debug is off |
| `dd` (`@dd`) | `@dd($var)` — no-op when debug is off |
| `@{{` | Literal `{{ }}` (not compiled) |

See the [Directives Reference](lte-directives.md) for what each directive does.

## File icon

`.lte` files show the Luany icon in the Explorer, editor tabs and Quick Open.

VS Code uses a language's own icon only when the active **File Icon Theme** has no specific icon for that file type. That is the case for the built-in **Seti** and **Minimal** themes. Themes that ship a complete icon set, such as *Material Icon Theme*, may keep showing their own generic icon for `.lte` files — switch the theme (**Preferences: File Icon Theme**) or configure that theme's file associations if you want the Luany icon.

## Error diagnostics

When you open or save a `.lte` file, the extension runs [`luany view:check`](view-check.md) in the project and underlines every problem at its exact line and column. The same problems are listed in the **Problems** panel (source `luany view:check`):

- unclosed, stray or crossed blocks (`@if` … `@endforeach`);
- misplaced `@else`, `@elseif` or `@empty`, and directives missing their arguments;
- PHP syntax errors inside `{{ }}`, `{!! !!}`, `@php` and directive arguments.

Diagnostics cover the whole project, not only the open file, and are replaced on every run. Runs are debounced (300 ms) and never overlap for the same project; a save made while a check is running triggers a fresh check once it finishes.

To validate on demand, open the Command Palette and run **Luany LTE: Validate views**. It reports `all views OK` or the number of problems in the status bar.

### Requirements

- luany/cli 1.4 or newer, installed globally (`composer global require luany/cli`) or in the project (`composer require --dev luany/cli`).
- `luany/lte` 1.2 or newer in the project.

The project root is the nearest folder above the file that contains `bootstrap/app.php`, or both `views/` and `composer.json`. The CLI is looked up in this order:

1. The `luanyLte.cliPath` setting.
2. `vendor/bin/luany` in the project (run with `php`).
3. `luany` on your `PATH`.

If no CLI is found, the extension shows a one-time warning with the install command. If `view:check` fails without results — an old CLI, an application that cannot boot, a timeout (30 s) — the existing diagnostics are kept and the reason is written to the **Luany LTE** output channel.

### Settings

| Setting | Default | Description |
|---|---|---|
| `luanyLte.validate.enabled` | `true` | Run validation on open and save. Turning it off clears the diagnostics. |
| `luanyLte.validate.bootApp` | `false` | Boot the application while validating, so directives registered by your service providers are known. Slower. |
| `luanyLte.cliPath` | `""` | Path to the `luany` executable. Empty: `vendor/bin/luany`, then `luany` on `PATH`. |

```json
{
    "luanyLte.validate.enabled": true,
    "luanyLte.validate.bootApp": true,
    "luanyLte.cliPath": "/home/me/.config/composer/vendor/bin/luany"
}
```

With `bootApp` off (the default), the extension passes `--no-app`: custom directives are treated as plain text, but structure and PHP syntax are still checked. Turn it on if your providers register directives and your `.env` lets the application boot.
