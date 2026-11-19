# Deployment

This page walks through putting a Luany application into production: environment settings, dependencies, caches, file permissions, web server configuration for Apache and Nginx, HTTPS behind proxies, and migrations.

## Requirements

- PHP 8.2 or newer with `pdo`, the PDO driver for your database (`pdo_sqlite`, `pdo_mysql` or `pdo_pgsql`), `mbstring`, `openssl` and `json`.
- Composer.
- A web server (Apache or Nginx) with PHP-FPM, or any server that can route every request to `public/index.php`.

Node.js is **not** needed in production — it only powers `luany dev`.

Run `luany doctor` on the server to check PHP, extensions, `.env`, `APP_KEY`, storage permissions and the database connection in one go (see [Commands](cli.md#doctor)).

## Environment

Create `.env` on the server (it is git-ignored) from `.env.example` and set at least:

```env
APP_NAME="My App"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com
APP_KEY=base64:...
APP_TIMEZONE=UTC

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=myapp
DB_USER=myapp
DB_PASS=secret
```

| Setting | Why |
|---|---|
| `APP_ENV=production` | Disables live reload and development-only behaviour |
| `APP_DEBUG=false` | Hides the debug error page (stack traces, file paths) and disables `@dump` / `@dd` |
| `APP_KEY` | Generate with `luany key:generate` if the file is new |
| `DB_CONNECTION` | Always set it explicitly |

Real environment variables take precedence over `.env` — both in the application and in the CLI — so container platforms can inject settings instead of shipping a file. `APP_ENV` and `APP_URL` are required: the skeleton's bootstrap throws if they are missing. See [Configuration](configuration.md).

Protect the file:

```bash
chmod 640 .env
```

## Installing dependencies

Install without development packages and with an optimized autoloader:

```bash
composer install --no-dev --optimize-autoloader --no-interaction
```

The skeleton already sets `optimize-autoloader` in `composer.json`. If you deploy from a build artifact, run the same command in the build step and ship `vendor/`.

## Database migrations

Run pending migrations on every deploy:

```bash
luany migrate
```

The CLI reads `DB_*` from the real environment first, then `.env`, so it targets the same database as the application. `luany` must be installed on the server (`composer global require luany/cli`) or added as a dependency.

> **Warning** Never run `luany migrate:fresh` in production — it drops every table. It asks for confirmation unless `--force` is given, and the prompt defaults to "No".

Check what ran with `luany migrate:status`; undo the last batch with `luany migrate:rollback`.

## Caches

### Views

LTE compiles templates to PHP in `storage/cache/views`. In production (`APP_DEBUG=false`) a template is recompiled only when its source file is newer than the compiled file, so new templates are picked up after a deploy. Clear the cache anyway to remove stale compiled files:

```bash
luany cache:clear
```

Enable OPcache in production; compiled templates are invalidated in OPcache when they are rewritten.

### Routes

Without a cache, the HTTP Kernel loads `routes/http.php` and every other `routes/*.php` file on each request; with OPcache enabled this is cheap. When `storage/cache/routes.php` exists, the Kernel loads the route table from it instead.

The CLI has no `route:cache` command. Generate the file in your deploy script, after `composer install`, with a script such as the `bin/route-cache.php` shown in [Routing](routing.md#using-the-cache-in-the-kernel):

```bash
php bin/route-cache.php
```

The cache requires controller or `Route::view()` actions — `Route::cache()` throws and lists any closure route. A stale cache keeps serving the old routes, so regenerate it on **every** deploy and never ship one built from different route files. Keep it out of version control — the skeleton's `.gitignore` excludes `/storage/cache/routes.php` since 1.4.3 (add the line in older projects).

## File permissions

The web server user (for example `www-data`) must be able to write to:

| Path | Used for |
|---|---|
| `storage/cache/views` | Compiled templates |
| `storage/logs` | Daily log files |
| `storage/sessions` | Session files (created on first use) |
| `storage/rate-limits` (or your path) | `FileRateLimiter`, if used |
| `database/` | SQLite only: the database file **and** its directory (SQLite writes a journal next to it) |

Everything else can be read-only:

```bash
sudo chown -R deploy:www-data /var/www/myapp
sudo find /var/www/myapp -type d -exec chmod 755 {} \;
sudo find /var/www/myapp -type f -exec chmod 644 {} \;
sudo chmod -R ug+rwX /var/www/myapp/storage
sudo chmod 640 /var/www/myapp/.env
```

## Web server

Point the document root at `public/`. Only `public/` may be reachable from the web: the project root contains `.env`, `vendor/`, `storage/` and your source code.

### Apache

The skeleton ships `public/.htaccess`. With `mod_rewrite` enabled and `AllowOverride All` for the directory, it:

- routes every request that is not an existing file or directory to `index.php`;
- passes the `Authorization` header to PHP;
- sets `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` and `Permissions-Policy` (with `mod_headers`);
- denies access to dotfiles and disables directory listings.

```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/myapp/public

    <Directory /var/www/myapp/public>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Uncomment the `Strict-Transport-Security` header in `.htaccess` once the site is HTTPS-only, and the `Content-Security-Policy` example after adapting it to your assets.

### Nginx

Nginx does not read `.htaccess`; translate its rules into the server block:

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/myapp/public;
    index index.php;

    charset utf-8;
    client_max_body_size 10m;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param HTTP_AUTHORIZATION $http_authorization;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    # Never serve dotfiles (.env, .git, ...)
    location ~ /\.(?!well-known) {
        deny all;
    }
}
```

Adjust the PHP-FPM socket to your PHP version.

## HTTPS and proxies

Serve the site over HTTPS only and redirect HTTP to HTTPS at the web server or load balancer.

If TLS terminates at a load balancer or reverse proxy, PHP sees plain HTTP from the proxy's address. Two settings fix that:

1. Trust the proxy, so `$request->ip()` and `$request->isSecure()` use `X-Forwarded-For` / `X-Forwarded-Proto` — for example in `AppServiceProvider::boot()`:

   ```php
   use Luany\Core\Http\Request;

   Request::setTrustedProxies(['10.0.0.0/8']);
   ```

2. Force secure session cookies — the automatic detection only looks at the server's own HTTPS flag:

   ```php
   // config/session.php
   return [
       'secure' => true,
   ];
   ```

Set `APP_URL` to the public `https://` URL so `url()` and `route()` generate correct links (`asset()` is root-relative; set `ASSET_URL` to serve assets from a CDN).

## Deployment checklist

- [ ] `.env` present with `APP_ENV=production`, `APP_DEBUG=false`, `APP_URL`, `APP_KEY` and explicit `DB_*` settings; `chmod 640`.
- [ ] `composer install --no-dev --optimize-autoloader`.
- [ ] Document root is `public/`; Apache `.htaccess` active or equivalent Nginx rules.
- [ ] `storage/` (and `database/` for SQLite) writable by the web server.
- [ ] `luany migrate` run.
- [ ] `luany cache:clear` run; OPcache enabled.
- [ ] Route cache regenerated (or deleted) if you use one.
- [ ] HTTPS enforced; trusted proxies and `secure` session cookie configured when behind a proxy.
- [ ] `luany doctor` passes.
- [ ] Logs in `storage/logs` monitored and rotated.

See the [Security](security.md) checklist for application-level items.
