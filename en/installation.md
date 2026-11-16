# Installation

This page covers the requirements, creating a new project with the Luany CLI, and running it locally.

## Requirements

| Requirement | Notes |
| --- | --- |
| PHP 8.2+ | With `pdo_sqlite` for the default setup, or `pdo_mysql` / `pdo_pgsql` |
| Composer | Installs the framework packages |
| Node.js | Only needed for live reload (`luany dev`) |

Run `luany doctor` at any time to check your environment and the health of a project.

## Installing the CLI

The `luany` command is installed once, globally, with Composer:

```bash
composer global require luany/cli
```

Make sure Composer's global `bin` directory is on your `PATH`:

```bash
# macOS / Linux
export PATH="$PATH:$HOME/.composer/vendor/bin"

# Windows
%USERPROFILE%\AppData\Roaming\Composer\vendor\bin
```

## Creating a project

```bash
luany new my-app
```

`luany new` asks its questions **before** Composer runs, so you can walk away while dependencies install:

```
  Database
   ›  1  sqlite     file-based, nothing to install (recommended to start)
      2  mysql      MySQL / MariaDB server
      3  pgsql      PostgreSQL server
  Choose 1-3 or a name [sqlite]:
  Initialize a git repository? [Y/n]:
```

- Drivers whose PDO extension is missing are tagged `(PHP extension missing)` in the list.
- For **MySQL** or **PostgreSQL** you are asked for host, port, database name, user and password. Defaults are `127.0.0.1`, port `3306` / `5432`, a database named after the project (lowercased, other characters replaced by `_`) and user `root` / `postgres`.
- The git question only appears when `git` is installed.

The command then runs `composer create-project luany/luany my-app` and configures the database:

| Database | What `luany new` does |
| --- | --- |
| SQLite | Writes `DB_CONNECTION=sqlite`, creates `database/database.sqlite` and runs the migrations — the app works immediately |
| MySQL / PostgreSQL | Writes the `DB_*` values to `.env`. Create the database yourself, then run `luany migrate` |

If the PDO extension for the chosen driver is missing, the command explains how to install it (apt, dnf, php.ini) and skips the migrations.

### Options

| Option | Description |
| --- | --- |
| `--database=sqlite\|mysql\|pgsql` | Choose the database without being asked |
| `--git` / `--no-git` | Initialize (or skip) a git repository |
| `--no-migrate` | Do not run the migrations for SQLite |
| `-n`, `--no-interaction` | Ask nothing: SQLite, migrated, no git (unless `--git`) |

```bash
luany new my-app --database=pgsql --git
luany new my-app -n
```

Project names may contain letters, digits, `.`, `_` and `-`, and must start with a letter or digit.

### Without the CLI

You can also create a project with Composer directly:

```bash
composer create-project luany/luany my-app
cd my-app
luany migrate
```

The skeleton's post-install script copies `.env.example` to `.env`, writes a fresh `APP_KEY` and creates the SQLite database file. Unlike `luany new`, it does not run the migrations.

### Versions

New projects start from skeleton 1.4.2 with `luany/framework` 1.4 and `luany/core` 1.4. To bring an existing project up to date:

```bash
composer update luany/framework luany/core
```

Projects created before skeleton 1.4.2 should also add `/storage/sessions/` to `.gitignore` and delete the empty `$dontReport` override in `app/Exceptions/Handler.php` — see [Error Handling](errors.md#reporting).

## Running the application

```bash
cd my-app
luany dev
```

`luany dev` starts PHP's built-in server on `http://localhost:8000` together with a file watcher that reloads the browser when you change views, PHP files, routes or config (CSS is injected without a full reload). It needs Node.js and the watcher dependencies:

```bash
npm install
```

Live reload is only active when `APP_ENV=development` and the request comes from the local machine. See [Dev Server & Live Reload](dev-server.md).

Without Node.js, use the plain PHP server:

```bash
luany serve                    # http://localhost:8000
luany serve localhost 8080     # custom host and port
```

## Web server configuration

Point your web server's document root at the `public/` directory — never at the project root, which contains `.env`.

**Apache** works out of the box: `public/.htaccess` routes every request that is not a real file to `index.php`, forwards the `Authorization` header, denies dotfiles and sets basic security headers. It has no hard-coded `RewriteBase`, so the app also works in a sub-directory.

**Nginx** needs the usual front-controller rule:

```nginx
root /var/www/my-app/public;
index index.php;

location / {
    try_files $uri $uri/ /index.php?$query_string;
}

location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    fastcgi_pass unix:/run/php/php8.2-fpm.sock;
}
```

Make `storage/` writable by the web server user: views are compiled to `storage/cache/views`, logs go to `storage/logs` and sessions to `storage/sessions`.

## Next steps

- [Configuration](configuration.md) — `.env` and `config/app.php`
- [Directory Structure](directory-structure.md) — where things live
- [Routing](routing.md) — add your first pages
- [make:feature](make-feature.md) — scaffold a complete CRUD feature in one command
