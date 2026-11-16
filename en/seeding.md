# Seeding

Seeders fill the database with initial or test data. They live in `database/seeders/` and run with `luany db:seed`.

## Writing seeders

Generate a seeder:

```bash
luany make:seeder UserSeeder
```

The `Seeder` suffix is added when missing. A seeder extends `Luany\Database\Seeder\Seeder` and implements `run(\PDO $pdo)`:

```php
<?php

use Luany\Database\Seeder\Seeder;

class UserSeeder extends Seeder
{
    public function run(\PDO $pdo): void
    {
        $stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (?, ?)');

        foreach ([['Ana', 'ana@example.com'], ['Bruno', 'bruno@example.com']] as $row) {
            $stmt->execute($row);
        }
    }
}
```

Seeders are plain classes in the global namespace; the runner loads every file in `database/seeders/` before running, so they need no autoloading.

## The DatabaseSeeder

`luany db:seed` runs `DatabaseSeeder`, which the skeleton provides. Chain other seeders with `call()`; they receive the same PDO and run in order:

```php
<?php

use Luany\Database\Seeder\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(\PDO $pdo): void
    {
        $this->call(
            UserSeeder::class,
            CategorySeeder::class,
            PostSeeder::class,
        );
    }
}
```

`call()` throws `\RuntimeException` when a class does not exist or does not extend `Seeder`. Inside a seeder the PDO is also available as `$this->pdo`.

## Using the query builder and models

The CLI runs seeders without booting the application, but it gives your models its own connection (`Model::setConnection()`), so models work as in the application. For the query builder, wrap the PDO:

```php
<?php

use App\Models\Post;
use Luany\Database\Connection;
use Luany\Database\QueryBuilder;
use Luany\Database\Seeder\Seeder;

class PostSeeder extends Seeder
{
    public function run(\PDO $pdo): void
    {
        $db = new QueryBuilder(Connection::fromPdo($pdo));
        $db->table('categories')->insert([
            ['name' => 'News',   'slug' => 'news'],
            ['name' => 'Guides', 'slug' => 'guides'],
        ]);

        foreach (range(1, 20) as $i) {
            Post::create(['title' => "Post {$i}", 'body' => 'Lorem ipsum…']);
        }
    }
}
```

Models apply casts, timestamps and events as usual. For idempotent seeders, use `Model::firstOrCreate()` or [`upsert()`](query-builder.md#upserts) so running them twice does not duplicate rows.

On SQLite the CLI enables foreign keys (`PRAGMA foreign_keys = ON`), like the application, so seed the referenced tables (categories, users) before the rows that point to them — `call()` runs seeders in the order given.

> **Note** The model connection is bound automatically by luany/cli 1.6 or newer (`db:seed`, `migrate:fresh --seed` and the other `migrate*` commands). With an older CLI, call `Model::setConnection(Connection::fromPdo($pdo))` at the start of `run()`.

## Running seeders

| Command | Description |
| --- | --- |
| `luany db:seed` | Run `DatabaseSeeder`. |
| `luany db:seed --class=UserSeeder` | Run one seeder (also `--class UserSeeder`). |
| `luany migrate:fresh --seed` | Rebuild the schema, then run `DatabaseSeeder`. |

> **Warning** `migrate:fresh` drops every table. Never run it against production data.

## Running from PHP

`SeederRunner` is the engine used by the CLI:

```php
use Luany\Database\Seeder\SeederRunner;

$runner = new SeederRunner(app('db'), base_path('database/seeders'));

$runner->run('DatabaseSeeder', function (string $class, string $status) {
    echo "[{$status}] {$class}\n";
});
```

`run(string $seederClass = 'DatabaseSeeder', ?callable $output = null)` loads every file in the directory, then runs the class; it throws `\RuntimeException` when the class is missing or is not a `Seeder`. The output callback is called once, for the top-level seeder, with status `seeded`.
