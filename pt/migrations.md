# Migrations e schema

As migrations são ficheiros PHP versionados que criam e alteram o schema da base de dados. Usam um schema builder portável que gera o DDL correto para SQLite, MySQL e PostgreSQL.

## Gerar migrations

```bash
luany make:migration create_posts_table
luany make:migration add_slug_to_posts_table
```

Os ficheiros são escritos em `database/migrations/` com um prefixo de data e hora (`2026_11_11_093000_create_posts_table.php`) e executados pela ordem do nome do ficheiro. O nome determina o stub usado:

| Padrão do nome | Stub |
| --- | --- |
| `create_{table}_table` | `create('{table}')` com `id()` e `timestamps()`; `down()` elimina a tabela |
| `…_to_{table}_table`, `…_from_…`, `…_in_…`, `…_on_…` | `table('{table}')` em `up()` e `down()` |
| qualquer outro | `up()` / `down()` vazios |

O [`make:feature`](make-feature.md) gera uma migration juntamente com o modelo, o controller e as vistas. Não é possível criar uma migration com um nome que já existe.

## Estrutura de uma migration

Uma migration devolve uma classe anónima que estende `Migration`, com `up()` e `down()` a receberem o PDO:

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

As classes anónimas evitam colisões de nomes. As classes com nome continuam a funcionar: o nome da classe é derivado do nome do ficheiro sem os segmentos numéricos iniciais (`2026_11_11_093000_create_posts_table.php` → `CreatePostsTable`); uma classe declarada duas vezes é reportada com uma mensagem clara.

`$this->schema($pdo)` é um atalho para `Schema::for($pdo)`, que também aceita uma `Connection`. Pode sempre executar SQL em bruto com `$pdo->exec(…)`.

## Executar migrations

| Comando | Descrição |
| --- | --- |
| `luany migrate` | Executa todas as migrations pendentes como um novo lote. |
| `luany migrate:rollback` | Reverte o último lote. |
| `luany migrate:status` | Lista as migrations, indicando se foram executadas ou estão pendentes, e o respetivo lote. |
| `luany migrate:fresh` | Elimina **todas** as tabelas e volta a executar todas as migrations. Pede confirmação; `--force` dispensa-a e `--seed` executa o `DatabaseSeeder` no fim. |

Os comandos leem as variáveis `DB_*` do ambiente e do `.env` (ver [Introdução](database.md#configuração)) sem arrancar a aplicação. O progresso é registado numa tabela `_migrations` (nome, lote, data e hora).

Em SQLite e PostgreSQL, cada migration é executada numa transação juntamente com o respetivo registo, por isso uma migration que falhe não deixa o schema a meio. O MySQL faz commit implícito do DDL; corrija e volte a executar manualmente uma migration MySQL que tenha falhado.

### Executar a partir de PHP

```php
use Luany\Database\Migration\MigrationRunner;

$runner = new MigrationRunner(app('db'), base_path('database/migrations'));

$runner->run(fn (string $name, string $status) => print("[{$status}] {$name}\n"));
$runner->rollback(steps: 2);
```

| Método | Devolve |
| --- | --- |
| `run(?callable $output = null)` | Número de migrations executadas |
| `rollback(?callable $output = null, int $steps = 1)` | Número de migrations revertidas |
| `reset(?callable $output = null)` | Reverte todos os lotes |
| `pending(): array` | Nomes ainda não executados |
| `status(): array` | `[['name' => …, 'ran' => bool, 'batch' => ?int], …]` |
| `dropAll(\PDO $pdo)` | Elimina todas as tabelas (usado por `migrate:fresh`) |

O callback de output recebe `(string $name, string $status)`, com o estado `migrated`, `rolled back` ou `nothing`.

## Tabelas

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
$schema->hasColumn('users', 'email');    // bool (sem distinção entre maiúsculas e minúsculas)
$schema->getColumnListing('users');      // ['id', 'name', …]
$schema->toSql('posts', fn (Blueprint $t) => $t->id());   // pré-visualização do DDL, list<string>
```

## Colunas

| Método | MySQL | PostgreSQL | SQLite |
| --- | --- | --- | --- |
| `id(string $name = 'id')`, `bigIncrements($name)` | `BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY` | `BIGSERIAL PRIMARY KEY` | `INTEGER PRIMARY KEY AUTOINCREMENT` |
| `increments($name)` | `INT UNSIGNED AUTO_INCREMENT PRIMARY KEY` | `SERIAL PRIMARY KEY` | igual ao anterior |
| `foreignId($name)` | `BIGINT UNSIGNED` | `BIGINT` | `INTEGER` |
| `string($name, int $length = 255)` | `VARCHAR(n)` | `VARCHAR(n)` | `VARCHAR(n)` |
| `char($name, int $length = 255)` | `CHAR(n)` | `CHAR(n)` | `CHAR(n)` |
| `text($name)` / `longText($name)` | `TEXT` / `LONGTEXT` | `TEXT` | `TEXT` |
| `uuid(string $name = 'uuid')` | `CHAR(36)` | `UUID` | `CHAR(36)` |
| `enum($name, array $values)` | `ENUM(…)` | `VARCHAR(255)` + `CHECK` | `VARCHAR(255)` + `CHECK` |
| `integer`, `bigInteger`, `smallInteger`, `tinyInteger` | `INT`, `BIGINT`, `SMALLINT`, `TINYINT` | `INTEGER`, `BIGINT`, `SMALLINT`, `SMALLINT` | `INTEGER` |
| `unsignedInteger`, `unsignedBigInteger` | `… UNSIGNED` | com sinal | `INTEGER` |
| `decimal($name, int $precision = 10, int $scale = 2)` | `DECIMAL(p, s)` | `NUMERIC(p, s)` | `NUMERIC` |
| `float` / `double` | `FLOAT` / `DOUBLE` | `REAL` / `DOUBLE PRECISION` | `REAL` |
| `boolean` | `TINYINT(1)` | `BOOLEAN` | `INTEGER` |
| `date`, `time` | `DATE`, `TIME` | `DATE`, `TIME(0)` | `DATE`, `TIME` |
| `dateTime`, `timestamp` | `DATETIME`, `TIMESTAMP` | `TIMESTAMP(0)` | `DATETIME` |
| `json` | `JSON` | `JSONB` | `TEXT` |
| `binary` | `BLOB` | `BYTEA` | `BLOB` |

Atalhos:

| Método | Acrescenta |
| --- | --- |
| `timestamps()` | `created_at` e `updated_at` que aceitam nulos (preenchidos pelos modelos com `$timestamps = true`) |
| `softDeletes(string $column = 'deleted_at')` | timestamp que aceita nulos, para [`SoftDeletes`](models.md#soft-deletes) |
| `morphs(string $name, bool $nullable = false)` | string `{name}_type`, inteiro grande sem sinal `{name}_id` e um índice composto |

As tabelas MySQL são criadas com `ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci`.

### Modificadores de colunas

```php
$table->string('email', 150)->unique();
$table->decimal('price', 10, 2)->default(0);
$table->timestamp('paid_at')->nullable();
$table->timestamp('seen_at')->useCurrent();
$table->string('code')->index();
$table->json('meta')->default(Schema::raw("'{}'"));
```

| Modificador | Efeito |
| --- | --- |
| `nullable(bool $value = true)` | Permite `NULL` (as colunas são `NOT NULL` por predefinição). |
| `default(mixed $value)` | Valor predefinido literal: escalar, `null`, bool, backed enum ou `Stringable`. As strings são citadas pelo PDO. |
| `useCurrent()` | `DEFAULT CURRENT_TIMESTAMP`. |
| `unsigned()` | `UNSIGNED` em MySQL; ignorado nos restantes. |
| `unique()` / `index()` | Índice de uma só coluna. |
| `primary()` | Faz parte da chave primária. |

`Schema::raw(string $sql)` passa uma expressão SQL como valor predefinido. Nunca a construa a partir de input do utilizador.

## Índices

```php
$table->index('status');
$table->index(['user_id', 'created_at']);
$table->unique(['team_id', 'email'], 'team_member_unique');
$table->primary(['user_id', 'role_id']);   // chave composta
```

Por predefinição, os nomes seguem o formato `{table}_{columns}_{type}`, por exemplo `posts_user_id_created_at_index`.

## Chaves estrangeiras

```php
$table->foreignId('user_id')->constrained();                     // users(id)
$table->foreignId('category_id')->nullable()->constrained()->nullOnDelete(); // categories(id)
$table->foreignId('author_id')->constrained('users')->cascadeOnDelete();

$table->foreign('team_id')->references('id')->on('teams')->onDelete('cascade');
```

`constrained(?string $table = null, string $column = 'id')` deduz a tabela a partir da coluna (`user_id` → `users`, `category_id` → `categories`). Chame `nullable()`, `index()`, `unique()` ou `default()` **antes** de `constrained()`.

| Método da `ForeignKeyDefinition` | Efeito |
| --- | --- |
| `references(string $column)`, `on(string $table)` | Coluna / tabela de destino. |
| `onDelete(string $action)`, `onUpdate(string $action)` | `CASCADE`, `RESTRICT`, `SET NULL`, `NO ACTION`, `SET DEFAULT`. |
| `cascadeOnDelete()`, `restrictOnDelete()`, `nullOnDelete()`, `cascadeOnUpdate()` | Atalhos. |
| `name(string $name)` | Nome da restrição (predefinição: `{table}_{columns}_foreign`). |

Os valores predefinidos são `ON DELETE RESTRICT ON UPDATE CASCADE`.

> **Nota** O SQLite só aplica as chaves estrangeiras quando `PRAGMA foreign_keys = ON` está definido na ligação. A ligação não o ativa automaticamente.

## Alterar tabelas

```php
$this->schema($pdo)->table('posts', function (Blueprint $table) {
    $table->string('subtitle')->nullable();           // acrescenta coluna
    $table->renameColumn('body', 'content');
    $table->dropColumn(['legacy_flag', 'old_slug']);
    $table->index('published');
    $table->dropIndex('posts_status_index');          // pelo nome …
    $table->dropUnique(['slug']);                     // … ou pelas colunas
    $table->dropForeign(['user_id']);                 // apenas MySQL / PostgreSQL
});
```

As eliminações e renomeações são executadas primeiro, e depois as adições. As novas colunas `NOT NULL` em tabelas existentes precisam de um `default()`.

## Limitações

| Não suportado | Alternativa |
| --- | --- |
| Alterar o tipo ou os modificadores de uma coluna (não há `change()`) | Acrescente uma nova coluna, copie os dados e elimine a antiga; ou use SQL em bruto. |
| Acrescentar uma coluna autoincrementada ou uma chave primária a uma tabela existente | Defina-a em `create()`. |
| SQLite: acrescentar ou eliminar uma chave estrangeira numa tabela existente | Recrie a tabela. Ambas as operações lançam `\LogicException`. |
| SQLite: `dropColumn()` / `renameColumn()` | Exigem SQLite 3.35 / 3.25 ou posterior. |
| `morphs()` com chaves UUID | Declare manualmente as colunas `{name}_type` e `{name}_id`. |
