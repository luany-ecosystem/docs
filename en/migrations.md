# Migrations & Schema

Migrations are versioned PHP files that create and change your database schema. They use a portable schema builder that generates the right DDL for SQLite, MySQL and PostgreSQL.

## Generating migrations

```bash
luany make:migration create_posts_table
luany make:migration add_slug_to_posts_table
```

Files are written to `database/migrations/` with a timestamp prefix (`2026_11_11_093000_create_posts_table.php`) and run in filename order. The name picks the stub:

| Name pattern | Stub |
| --- | --- |
| `create_{table}_table` | `create('{table}')` with `id()` and `timestamps()`; `down()` drops the table |
| `…_to_{table}_table`, `…_from_…`, `…_in_…`, `…_on_…` | `table('{table}')` in `up()` and `down()` |
| anything else | empty `up()` / `down()` |

[`make:feature`](make-feature.md) generates a migration together with the model, controller and views. Creating a migration whose name already exists is refused.

## Migration structure

A migration returns an anonymous class extending `Migration`, with `up()` and `down()` receiving the PDO:

```php
<?php

use Luany\Database\Migration\Migration;
use Luany\Database\Schema\Blueprint;

return new class extends Migration
{
    public function up(\PDO $pdo): void
    {
        $this->schema($pdo)->create('posts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('title');
            $table->string('slug')->unique();
            $table->text('body')->nullable();
            $table->boolean('published')->default(false);
            $table->timestamps();
            $table->softDeletes();
        });
    }

    public function down(\PDO $pdo): void
    {
        $this->schema($pdo)->dropIfExists('posts');
    }
};
```

Anonymous classes avoid name collisions. Named classes still work: the class name is derived from the file name without its leading digit segments (`2026_11_11_093000_create_posts_table.php` → `CreatePostsTable`); a class declared twice is reported clearly.

`$this->schema($pdo)` is shorthand for `Schema::for($pdo)`, which also accepts a `Connection`. Raw SQL is always possible with `$pdo->exec(…)`.

## Running migrations

| Command | Description |
| --- | --- |
| `luany migrate` | Run all pending migrations as a new batch. |
| `luany migrate:rollback` | Roll back the last batch. |
| `luany migrate:rollback --step=N` | Roll back the last N migrations, newest first, across batches (luany/cli 1.6 and luany/database 1.8 or newer). |
| `luany migrate:status` | List migrations with ran / pending and their batch. |
| `luany migrate:fresh` | Drop **all** tables and re-run every migration. Asks for confirmation; `--force` skips it, `--seed` runs `DatabaseSeeder` afterwards. |

The commands read `DB_*` from the environment and `.env` (see [Getting Started](database.md#configuration)) without booting the application. Progress is tracked in a `_migrations` table (name, batch, timestamp).

On SQLite and PostgreSQL each migration runs in a transaction together with its bookkeeping, so a failing migration leaves no partial schema. MySQL commits DDL implicitly; fix and re-run a failed MySQL migration by hand.

### Running from PHP

```php
use Luany\Database\Migration\MigrationRunner;

$runner = new MigrationRunner(app('db'), base_path('database/migrations'));

$runner->run(fn (string $name, string $status) => print("[{$status}] {$name}\n"));
$runner->rollback(steps: 2);   // last two batches
$runner->rollbackLast(3);       // last three migrations, any batch
```

| Method | Returns |
| --- | --- |
| `run(?callable $output = null)` | Number of migrations run |
| `rollback(?callable $output = null, int $steps = 1)` | Number rolled back (the last `$steps` batches) |
| `rollbackLast(int $migrations, ?callable $output = null)` | Number rolled back (the last N migrations, newest first, across batches; throws `\InvalidArgumentException` below 1) |
| `reset(?callable $output = null)` | Roll back every batch |
| `pending(): array` | Names not yet run |
| `status(): array` | `[['name' => …, 'ran' => bool, 'batch' => ?int], …]` |
| `dropAll(\PDO $pdo)` | Drop every table (used by `migrate:fresh`) |

The output callback receives `(string $name, string $status)` with status `migrated`, `rolled back` or `nothing`.

## Tables

```php
use Luany\Database\Schema\Schema;

$schema = Schema::for($pdo);

$schema->create('tags', fn (Blueprint $t) => $t->id());
$schema->createIfNotExists('tags', fn (Blueprint $t) => $t->id());
$schema->table('tags', fn (Blueprint $t) => $t->string('color')->nullable());
$schema->rename('tags', 'labels');
$schema->drop('labels');
$schema->dropIfExists('labels');

$schema->hasTable('users');              // bool
$schema->hasColumn('users', 'email');    // bool (case-insensitive)
$schema->getColumnListing('users');      // ['id', 'name', …]
$schema->toSql('posts', fn (Blueprint $t) => $t->id());   // preview DDL, list<string>
```

## Columns

| Method | MySQL | PostgreSQL | SQLite |
| --- | --- | --- | --- |
| `id(string $name = 'id')`, `bigIncrements($name)` | `BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY` | `BIGSERIAL PRIMARY KEY` | `INTEGER PRIMARY KEY AUTOINCREMENT` |
| `increments($name)` | `INT UNSIGNED AUTO_INCREMENT PRIMARY KEY` | `SERIAL PRIMARY KEY` | same as above |
| `foreignId($name)` | `BIGINT UNSIGNED` | `BIGINT` | `INTEGER` |
| `string($name, int $length = 255)` | `VARCHAR(n)` | `VARCHAR(n)` | `VARCHAR(n)` |
| `char($name, int $length = 255)` | `CHAR(n)` | `CHAR(n)` | `CHAR(n)` |
| `text($name)` / `longText($name)` | `TEXT` / `LONGTEXT` | `TEXT` | `TEXT` |
| `uuid(string $name = 'uuid')` | `CHAR(36)` | `UUID` | `CHAR(36)` |
| `enum($name, array $values)` | `ENUM(…)` | `VARCHAR(255)` + `CHECK` | `VARCHAR(255)` + `CHECK` |
| `integer`, `bigInteger`, `smallInteger`, `tinyInteger` | `INT`, `BIGINT`, `SMALLINT`, `TINYINT` | `INTEGER`, `BIGINT`, `SMALLINT`, `SMALLINT` | `INTEGER` |
| `unsignedInteger`, `unsignedBigInteger` | `… UNSIGNED` | signed | `INTEGER` |
| `decimal($name, int $precision = 10, int $scale = 2)` | `DECIMAL(p, s)` | `NUMERIC(p, s)` | `NUMERIC` |
| `float` / `double` | `FLOAT` / `DOUBLE` | `REAL` / `DOUBLE PRECISION` | `REAL` |
| `boolean` | `TINYINT(1)` | `BOOLEAN` | `INTEGER` |
| `date`, `time` | `DATE`, `TIME` | `DATE`, `TIME(0)` | `DATE`, `TIME` |
| `dateTime`, `timestamp` | `DATETIME`, `TIMESTAMP` | `TIMESTAMP(0)` | `DATETIME` |
| `json` | `JSON` | `JSONB` | `TEXT` |
| `binary` | `BLOB` | `BYTEA` | `BLOB` |

Shortcuts:

| Method | Adds |
| --- | --- |
| `timestamps()` | nullable `created_at` and `updated_at` (filled by models with `$timestamps = true`) |
| `softDeletes(string $column = 'deleted_at')` | nullable timestamp for [`SoftDeletes`](models.md#soft-deletes) |
| `morphs(string $name, bool $nullable = false)` | `{name}_type` string, `{name}_id` unsigned big integer, and a composite index |

MySQL tables are created with `ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci`.

### Column modifiers

```php
$table->string('email', 150)->unique();
$table->decimal('price', 10, 2)->default(0);
$table->timestamp('paid_at')->nullable();
$table->timestamp('seen_at')->useCurrent();
$table->string('code')->index();
$table->json('meta')->default(Schema::raw("'{}'"));
```

| Modifier | Effect |
| --- | --- |
| `nullable(bool $value = true)` | Allow `NULL` (columns are `NOT NULL` by default). |
| `default(mixed $value)` | Literal default: scalar, `null`, bool, backed enum or `Stringable`. Strings are quoted by PDO. |
| `useCurrent()` | `DEFAULT CURRENT_TIMESTAMP`. |
| `unsigned()` | `UNSIGNED` on MySQL; ignored elsewhere. |
| `unique()` / `index()` | Single-column index. |
| `primary()` | Part of the primary key. |

`Schema::raw(string $sql)` passes an SQL expression as a default. Never build one from user input.

## Indexes

```php
$table->index('status');
$table->index(['user_id', 'created_at']);
$table->unique(['team_id', 'email'], 'team_member_unique');
$table->primary(['user_id', 'role_id']);   // composite key
```

Names default to `{table}_{columns}_{type}`, e.g. `posts_user_id_created_at_index`.

## Foreign keys

```php
$table->foreignId('user_id')->constrained();                     // users(id)
$table->foreignId('category_id')->nullable()->constrained()->nullOnDelete(); // categories(id)
$table->foreignId('author_id')->constrained('users')->cascadeOnDelete();

$table->foreign('team_id')->references('id')->on('teams')->onDelete('cascade');
```

`constrained(?string $table = null, string $column = 'id')` derives the table from the column (`user_id` → `users`, `category_id` → `categories`). Call `nullable()`, `index()`, `unique()` or `default()` **before** `constrained()`.

| `ForeignKeyDefinition` method | Effect |
| --- | --- |
| `references(string $column)`, `on(string $table)` | Target column / table. |
| `onDelete(string $action)`, `onUpdate(string $action)` | `CASCADE`, `RESTRICT`, `SET NULL`, `NO ACTION`, `SET DEFAULT`. |
| `cascadeOnDelete()`, `restrictOnDelete()`, `nullOnDelete()`, `cascadeOnUpdate()` | Shortcuts. |
| `name(string $name)` | Constraint name (default `{table}_{columns}_foreign`). |

Defaults are `ON DELETE RESTRICT ON UPDATE CASCADE`.

> **Note** On SQLite, `Connection::make()` and the `luany` commands turn on `PRAGMA foreign_keys = ON`, so constraints and `ON DELETE` actions are enforced as on MySQL and PostgreSQL. Opt out with `'foreign_keys' => false` (see [Creating a connection](database.md#creating-a-connection)). `migrate:fresh` switches the pragma off while it drops the tables.

## Altering tables

```php
$this->schema($pdo)->table('posts', function (Blueprint $table) {
    $table->string('subtitle')->nullable();           // add column
    $table->renameColumn('body', 'content');
    $table->dropColumn(['legacy_flag', 'old_slug']);
    $table->index('published');
    $table->dropIndex('posts_status_index');          // by name …
    $table->dropUnique(['slug']);                     // … or by columns
    $table->dropForeign(['user_id']);                 // MySQL / PostgreSQL only
});
```

Drops and renames run first, then additions. New `NOT NULL` columns on existing tables need a `default()`.

## Limitations

| Not supported | Workaround |
| --- | --- |
| Changing a column's type or modifiers (no `change()`) | Add a new column, copy data, drop the old one; or raw SQL. |
| Adding an auto-increment column or a primary key to an existing table | Define it in `create()`. |
| SQLite: adding or dropping a foreign key on an existing table | Recreate the table. Both throw `\LogicException`. |
| SQLite: `dropColumn()` / `renameColumn()` | Require SQLite 3.35 / 3.25 or newer. |
| `morphs()` with UUID keys | Declare `{name}_type` and `{name}_id` columns yourself. |
