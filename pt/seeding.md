# Seeders

Os seeders preenchem a base de dados com dados iniciais ou de teste. Ficam em `database/seeders/` e são executados com `luany db:seed`.

## Escrever seeders

Gere um seeder:

```bash
luany make:seeder UserSeeder
```

O sufixo `Seeder` é acrescentado quando falta. Um seeder estende `Luany\Database\Seeder\Seeder` e implementa `run(\PDO $pdo)`:

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

Os seeders são classes simples no namespace global; o runner carrega todos os ficheiros de `database/seeders/` antes de os executar, por isso não precisam de autoloading.

## O DatabaseSeeder

`luany db:seed` executa o `DatabaseSeeder`, incluído no skeleton. Encadeie outros seeders com `call()`; recebem o mesmo PDO e são executados por ordem:

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

`call()` lança `\RuntimeException` quando uma classe não existe ou não estende `Seeder`. Dentro de um seeder, o PDO também está disponível como `$this->pdo`.

## Usar o query builder e os modelos

A CLI executa os seeders sem arrancar a aplicação, mas entrega aos modelos a sua própria ligação (`Model::setConnection()`), por isso os modelos funcionam como na aplicação. Para o query builder, envolva o PDO:

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

Os modelos aplicam casts, timestamps e eventos normalmente. Para seeders idempotentes, use `Model::firstOrCreate()` ou [`upsert()`](query-builder.md#upserts), para que executá-los duas vezes não duplique linhas.

No SQLite, a CLI ativa as chaves estrangeiras (`PRAGMA foreign_keys = ON`), tal como a aplicação, por isso preencha primeiro as tabelas referenciadas (categorias, utilizadores) e só depois as linhas que apontam para elas — `call()` executa os seeders pela ordem indicada.

> **Nota** A ligação dos modelos é registada automaticamente pela luany/cli 1.6 ou mais recente (`db:seed`, `migrate:fresh --seed` e os restantes comandos `migrate*`). Com uma CLI mais antiga, chame `Model::setConnection(Connection::fromPdo($pdo))` no início de `run()`.

## Executar seeders

| Comando | Descrição |
| --- | --- |
| `luany db:seed` | Executa o `DatabaseSeeder`. |
| `luany db:seed --class=UserSeeder` | Executa um único seeder (também `--class UserSeeder`). |
| `luany migrate:fresh --seed` | Reconstrói o schema e depois executa o `DatabaseSeeder`. |

> **Aviso** `migrate:fresh` elimina todas as tabelas. Nunca o execute sobre dados de produção.

## Executar a partir de PHP

O `SeederRunner` é o motor usado pela CLI:

```php
use Luany\Database\Seeder\SeederRunner;

$runner = new SeederRunner(app('db'), base_path('database/seeders'));

$runner->run('DatabaseSeeder', function (string $class, string $status) {
    echo "[{$status}] {$class}\n";
});
```

`run(string $seederClass = 'DatabaseSeeder', ?callable $output = null)` carrega todos os ficheiros da pasta e depois executa a classe; lança `\RuntimeException` quando a classe não existe ou não é um `Seeder`. O callback de output é chamado uma única vez, para o seeder de topo, com o estado `seeded`.
