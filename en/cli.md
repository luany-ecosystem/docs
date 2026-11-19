# Commands

The `luany` command-line tool (package `luany/cli`) creates projects, runs the development server, generates code and manages the database. This page lists every command, its arguments and options.

## Installation

Install the CLI once, globally, with Composer:

```bash
composer global require luany/cli
```

Make sure Composer's global `bin` directory is on your `PATH`. Its location depends on your system; ask Composer for it:

```bash
composer global config bin-dir --absolute
# e.g. add to ~/.bashrc or ~/.zshrc:
export PATH="$PATH:$HOME/.config/composer/vendor/bin"
```

Run `luany` (or `luany list`) without arguments to see all commands:

```bash
luany
```

### Project detection

Most commands must run inside a Luany project. The CLI walks up from the current directory until it finds a `composer.json` that requires `luany/framework` or `luany/core`, so you can run commands from any sub-directory.

| Situation | Result |
|---|---|
| Not inside a Luany project | `This command must be run inside a Luany project.` (exit 1) |
| Project found, `vendor/luany/framework` missing | Asks you to run `composer install` (exit 1) |

`new`, `doctor`, `about` and `list` work anywhere.

## Command overview

| Command | Description |
|---|---|
| `new` | Create a new Luany project |
| `serve` | Start the built-in PHP development server |
| `dev` | Start the Luany Dev Engine (PHP server + live reload) |
| `make:controller` | Scaffold a controller |
| `make:model` | Scaffold a model |
| `make:migration` | Generate a timestamped migration |
| `make:seeder` | Scaffold a seeder |
| `make:middleware` | Scaffold a middleware |
| `make:provider` | Scaffold a service provider |
| `make:view` | Create an LTE view (page, component or layout) |
| `make:request` | Scaffold a form request validation class |
| `make:test` | Scaffold a PHPUnit test |
| `make:feature` | Scaffold a complete CRUD feature |
| `migrate` | Run all pending migrations |
| `migrate:rollback` | Roll back the last migration batch (`--step=N`: the last N migrations) |
| `migrate:status` | Show the status of all migrations |
| `migrate:fresh` | Drop all tables and re-run all migrations |
| `db:seed` | Run database seeders |
| `route:list` | List all registered routes |
| `route:cache` | Cache the route table for production |
| `route:clear` | Delete the route cache |
| `view:check` | Validate all LTE views without rendering |
| `key:generate` | Generate and set `APP_KEY` in `.env` |
| `cache:clear` | Clear the compiled view cache |
| `doctor` | Check the environment and project health |
| `about` | Show project and package versions |
| `list` | List all available commands |

## Creating projects

```bash
luany new blog
```

`luany new` runs `composer create-project luany/luany <name>` and then configures the project. In an interactive terminal it asks every question **before** Composer starts, so you can walk away while dependencies install:

1. **Database** — a numbered menu: `sqlite` (default, nothing to install), `mysql` or `pgsql`. A driver whose PHP PDO extension is missing is marked `(PHP extension missing)`.
2. **Credentials** (MySQL / PostgreSQL only) — host, port, database name, user and password, with defaults.
3. **Git** — `Initialize a git repository? [Y/n]`, only when `git` is installed.

After Composer finishes, the CLI writes the `DB_*` settings to `.env`. With SQLite it also creates `database/database.sqlite` and runs the migrations, so the project is ready for `luany dev`.

| Option | Description |
|---|---|
| `--database=sqlite\|mysql\|pgsql` | Choose the database without the prompt |
| `--git` / `--no-git` | Initialize (or not) a git repository without the prompt |
| `--no-migrate` | Do not run the migrations after creating a SQLite project |
| `-n`, `--no-interaction` | Never prompt: SQLite, no git, default credentials |

```bash
# Non-interactive PostgreSQL project with git
luany new shop --database=pgsql --git -n
```

Without prompts, MySQL and PostgreSQL get default credentials: host `127.0.0.1`, port `3306` / `5432`, user `root` / `postgres`, an empty password and a database named after the project (lower-cased, non-alphanumeric characters replaced by `_`). Create that database, check `.env`, then run `luany migrate`.

Project names may contain letters, digits, `.`, `_` and `-`, and must start with a letter or digit.

### Missing PDO drivers

If the PHP extension for the chosen driver is missing, `luany new` (and `migrate*`, `db:seed`, `doctor`) explains how to install it instead of failing with PDO's "could not find driver":

```bash
PHP's pdo_sqlite extension is not installed (needed for DB_CONNECTION=sqlite).
     Ubuntu / Debian:  sudo apt install php8.3-sqlite3
     Fedora / RHEL:    sudo dnf install php-pdo
     Windows:          enable extension=pdo_sqlite in php.ini
     macOS (Homebrew): included in the php formula
```

In that case the automatic migration is skipped and `luany migrate` stays in the next steps.

## Development servers

| Command | Description |
|---|---|
| `luany serve [host] [port]` | PHP built-in server on `public/` (default `localhost 8000`) |
| `luany dev [host] [port] [ws-port]` | PHP server plus file watcher and browser live reload (default `localhost 8000 35729`) |

See [Dev Server & Live Reload](dev-server.md) for details.

## Generators

All generators validate names before writing anything. Class names accept StudlyCase, `snake_case` or `kebab-case` input (`blog_post` and `blog-post` both become `BlogPost`). Paths such as `../x`, names with spaces or names starting with a digit are rejected.

| Command | Creates | Suffix added | Sub-directories |
|---|---|---|---|
| `make:controller Post` | `app/Controllers/PostController.php` | `Controller` | Yes: `Admin/Post` |
| `make:model Post` | `app/Models/Post.php` (table `posts`) | — | No |
| `make:middleware Auth` | `app/Http/Middleware/AuthMiddleware.php` | `Middleware` | Yes |
| `make:provider Mail` | `app/Providers/MailServiceProvider.php` | `ServiceProvider` | No |
| `make:request StoreUser` | `app/Http/Requests/StoreUserRequest.php` | `Request` | Yes |
| `make:test UserController` | `tests/UserControllerTest.php` | `Test` | Yes |
| `make:seeder User` | `database/seeders/UserSeeder.php` | `Seeder` | No |

If the target file already exists, the generator leaves it untouched and prints a warning (`make:seeder` exits with an error instead).

Controllers generated in sub-directories (`Admin/Post` → namespace `App\Controllers\Admin`) import the base `App\Controllers\Controller`.

### make:migration

```bash
luany make:migration create_posts_table
luany make:migration add_slug_to_posts_table
```

The name must be `snake_case` and start with a letter. The file is written to `database/migrations/{Y_m_d_His}_{name}.php` as an anonymous-class migration using the schema builder. The template depends on the name:

| Name pattern | Generated `up()` / `down()` |
|---|---|
| `create_{table}_table` | `create('{table}', …)` with `id()` and `timestamps()` / `dropIfExists('{table}')` |
| `…_to_{table}_table`, `…_from_{table}_table`, `…_in_…`, `…_on_…` | `table('{table}', …)` in both methods |
| anything else | empty `up()` and `down()` |

Two migrations with the same name are refused: they would declare the same class. See [Migrations & Schema](migrations.md).

### make:view

```bash
luany make:view pages.about
luany make:view components.card component
luany make:view layouts.admin layout
```

The first argument is a dotted view name (segments of letters, digits, `_` or `-`); the file is created under `views/`. The optional second argument picks the template: `page` (default, extends `layouts.main`), `component` or `layout`.

### make:feature

`luany make:feature Product name:string price:decimal` generates a model, migration, controller, four views and a routes file in one go. It has its own page: [make:feature](make-feature.md).

## Database commands

| Command | Description |
|---|---|
| `luany migrate` | Run all pending migrations in `database/migrations` |
| `luany migrate:rollback` | Roll back the last batch |
| `luany migrate:rollback --step=N` | Roll back the last N migrations (newest first, across batches) |
| `luany migrate:status` | Table of migrations: ran or not, and batch number |
| `luany migrate:fresh` | Drop **all** tables, then run every migration |
| `luany db:seed` | Run `DatabaseSeeder` from `database/seeders` |

| Option | Command | Description |
|---|---|---|
| `--force` | `migrate:fresh` | Skip the confirmation prompt (required in scripts and CI) |
| `--seed` | `migrate:fresh` | Run `DatabaseSeeder` after migrating |
| `--step=N` / `--step N` | `migrate:rollback` | Roll back the last N migrations instead of the last batch (needs luany/database 1.8 or newer) |
| `--class=Name` / `--class Name` | `db:seed` | Run a specific seeder instead of `DatabaseSeeder` |

`migrate:fresh` asks `This drops ALL tables. Continue? [y/N]` and aborts by default — also when input is not a terminal, so always pass `--force` in automation. Without `--step`, `migrate:rollback` rolls back one whole batch; with an older luany/database, `--step` stops with a hint to run `composer update luany/database`.

```bash
luany migrate:fresh --seed --force
luany migrate:rollback --step=2
luany db:seed --class=UserSeeder
```

### Connection settings

Database commands read the same settings as the application, **without booting it**:

| Variable | Used for | CLI default |
|---|---|---|
| `DB_CONNECTION` | `sqlite`, `mysql` or `pgsql` | `sqlite` |
| `DB_NAME` | Database name, or the SQLite file path (relative to the project root) | `database/database.sqlite` for SQLite, otherwise `luany` |
| `DB_HOST` / `DB_PORT` | Server address | `127.0.0.1` / `3306` (`5432` for pgsql) |
| `DB_USER` / `DB_PASS` | Credentials | `root` (`postgres` for pgsql) / empty |

Values are resolved in this order:

1. Real environment variables (`$_ENV`, `$_SERVER`, `getenv()`) — Docker, CI and PaaS settings win.
2. The project's `.env` file.
3. The defaults above.

This is the same precedence the application uses (its `.env` loader never overwrites existing variables), so the CLI always talks to the same database as the app.

> **Note** The CLI defaults match the skeleton's `DatabaseServiceProvider` (`sqlite`, file `database/database.sqlite`). Projects on MySQL or PostgreSQL must set `DB_CONNECTION` in `.env`, as the application requires anyway. On SQLite the commands enable `PRAGMA foreign_keys = ON`, like the application, and `db:seed` / `migrate*` give your models the CLI's connection, so models work in seeders and migrations (see [Seeding](seeding.md#using-the-query-builder-and-models)).

If `.env` exists but cannot be read, the command stops with a permissions hint. If there is no `.env` and no `DB_*` variable in the environment, it asks you to run `luany key:generate`.

## Routes and views

### route:list

```bash
luany route:list
```

Loads `routes/http.php` and then every other `routes/*.php` file (alphabetically) without booting the framework, and prints a table of methods, URIs, actions (`PostController@show` or `Closure`) and names.

### route:cache / route:clear

```bash
luany route:cache
luany route:clear
```

`route:cache` boots the application like a request, reads every `routes/*.php` file and writes the route table to `storage/cache/routes.php` (the Kernel's `routeCachePath()`), which the HTTP Kernel then loads instead of the route files. It always rebuilds from the route files and exits with code 1, listing the routes, when an action is a closure. `route:clear` deletes the file. See [Routing](routing.md#using-the-cache-in-the-kernel).

### view:check

```bash
luany view:check
```

Validates every `.lte` view — block structure, required directive arguments and PHP syntax — without rendering. Exits with code 1 when a view has errors. See [view:check](view-check.md).

## Maintenance

### key:generate

```bash
luany key:generate
```

Creates `.env` from `.env.example` if needed and writes a new random `APP_KEY` (`base64:` + 32 random bytes). The command overwrites an existing key without asking. Projects created with `luany new` or `composer create-project` already have a key.

### cache:clear

```bash
luany cache:clear
```

Deletes the compiled templates in `storage/cache/views` and prints how many files were removed.

### doctor

```bash
luany doctor
```

Checks the environment, and — inside a project — the project's health:

| Scope | Checks |
|---|---|
| Global | PHP ≥ 8.2; extensions `pdo`, the PDO driver of `DB_CONNECTION` (`pdo_sqlite` by default), `mbstring`, `openssl`, `json`; Composer; `luany/cli` version |
| Project | `.env`, `vendor/`, `vendor/luany/framework`, `database/migrations`, `APP_KEY`, `bootstrap/app.php`, `public/index.php`, `config/app.php` |
| Storage | `storage/cache/views` and `storage/logs` are writable |
| Database | PDO driver installed, connection works (`from environment` when `DB_*` came from real variables), `_migrations` table exists |

### about

```bash
luany about
```

Shows the application name, environment, debug flag and URL (real environment variables first, then `.env`, like the framework's `env()`), plus the installed versions of PHP, `luany/framework`, `luany/core`, `luany/lte` (from `composer.lock`) and `luany/cli`.

## Exit codes

Commands exit with `0` on success and a non-zero code on errors: `1` for an unknown command, an invalid name or option, a command run outside a project or views with errors; an unexpected exception (for example a failing migration) ends with PHP's own non-zero code. Use them to fail CI jobs.
