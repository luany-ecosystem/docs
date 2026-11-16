# Introdução

O `luany/database` fornece um wrapper para ligações PDO, um query builder fluente, uma camada de modelos Active Record com relações, um schema builder portável, migrations e seeders. Suporta **SQLite**, **MySQL** e **PostgreSQL**.

## Configuração

Um projeto novo usa SQLite logo à partida — não é preciso nenhum servidor. A ligação é configurada no `.env`:

```env
# sqlite (sem servidor) | mysql | pgsql
DB_CONNECTION=sqlite
DB_NAME=database/database.sqlite
```

Para MySQL ou PostgreSQL:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=luany
DB_USER=root
DB_PASS=secret
```

| Variável | Predefinição | Notas |
| --- | --- | --- |
| `DB_CONNECTION` | `sqlite` | `sqlite`, `mysql` ou `pgsql`. |
| `DB_NAME` | `database/database.sqlite` (SQLite), `luany` | Os caminhos SQLite são relativos à raiz do projeto; `:memory:` é aceite. |
| `DB_HOST` | `127.0.0.1` | |
| `DB_PORT` | `3306` / `5432` | Escolhida conforme o driver. |
| `DB_USER` | `root` (MySQL) | |
| `DB_PASS` | vazia | |

> **Nota** Desde a luany/cli 1.6, os comandos de base de dados do `luany` (`migrate*`, `db:seed`, `doctor`) usam o mesmo valor predefinido, `sqlite`. Os projetos em MySQL ou PostgreSQL têm de definir `DB_CONNECTION` no `.env`.

O `App\Providers\DatabaseServiceProvider` do skeleton lê estes valores e regista:

| Binding no container | Valor |
| --- | --- |
| `connection` | Um singleton `Luany\Database\Connection`, aberto de forma preguiçosa na primeira query. |
| `db` | O `\PDO` em bruto (usado pelas migrations e pelas regras de [validação](validation.md) `unique` / `exists`). |

No `boot()`, chama `Model::setConnection(fn () => $app->make('connection'))`, para que todos os [modelos](models.md) partilhem a ligação.

## Criar uma ligação

Fora do skeleton, crie uma ligação com `Connection::make()`:

```php
use Luany\Database\Connection;

$connection = Connection::make([
    'driver'   => 'mysql',       // mysql (predefinido) | pgsql | sqlite
    'host'     => '127.0.0.1',
    'port'     => '3306',
    'database' => 'shop',
    'username' => 'root',
    'password' => 'secret',
    'charset'  => 'utf8mb4',     // apenas MySQL
]);

$sqlite = Connection::make(['driver' => 'sqlite', 'database' => __DIR__ . '/app.sqlite']);
$memory = Connection::make(['driver' => 'sqlite']);   // base de dados vazia → :memory:
```

O PDO é criado com exceções ativas, `FETCH_ASSOC`, prepared statements nativos e sem conversão dos valores lidos para string. Uma ligação falhada lança `\RuntimeException("Database connection failed: …")`; um driver desconhecido lança `\InvalidArgumentException`.

As ligações SQLite executam também `PRAGMA foreign_keys = ON`, pelo que as chaves estrangeiras e as ações `ON DELETE` / `ON UPDATE` são aplicadas tal como no MySQL e no PostgreSQL (luany/database 1.8 ou mais recente). Isto aplica-se também à ligação do skeleton. Passe `'foreign_keys' => false` para desativar:

```php
$legacy = Connection::make(['driver' => 'sqlite', 'database' => 'legacy.sqlite', 'foreign_keys' => false]);
```

`Connection::fromPdo(\PDO $pdo)` envolve um PDO já existente — útil em testes. As definições do PDO não são alteradas, incluindo o pragma `foreign_keys` do SQLite.

| Método | Descrição |
| --- | --- |
| `getPdo(): \PDO` | O PDO subjacente. |
| `getDriverName(): string` | `mysql`, `pgsql` ou `sqlite`. |
| `getGrammar(): Grammar` | Gramática de citação de identificadores para o driver. |
| `execute(string $sql, array $bindings = []): \PDOStatement` | Prepara e executa. Em MySQL e SQLite, os inteiros são associados como inteiros e os booleanos como `0`/`1`. |
| `lastInsertId(): string` | Último id inserido. |
| `inTransaction(): bool` | Indica se há uma transação PDO aberta. |

## Executar queries

O SQL em bruto passa por `execute()` com placeholders:

```php
$stmt = $connection->execute('SELECT * FROM users WHERE email = ?', [$email]);
$user = $stmt->fetch();
```

A maior parte do código usa o [query builder](query-builder.md) ou os [modelos](models.md):

```php
use Luany\Database\QueryBuilder;

$db = new QueryBuilder(app('connection'));

$active = $db->table('users')->where('active', '=', 1)->orderBy('name')->get();
```

## Transações

`transaction()` faz commit quando o callback termina e rollback quando este lança uma exceção; o callback recebe a ligação e o seu valor de retorno é devolvido:

```php
$orderId = $connection->transaction(function (Connection $db) use ($cart) {
    $db->execute('INSERT INTO orders (total) VALUES (?)', [$cart->total()]);
    $id = (int) $db->lastInsertId();

    foreach ($cart->items() as $item) {
        $db->execute('UPDATE products SET stock = stock - ? WHERE id = ?', [$item->qty, $item->id]);
    }

    return $id;
});
```

A exceção original é sempre relançada — uma falha durante o rollback nunca a esconde.

### Transações aninhadas

As chamadas aninhadas usam savepoints (`luany_sp_2`, `luany_sp_3`, …). Uma falha interior que o callback exterior apanhe desfaz apenas o trabalho interior:

```php
$connection->transaction(function (Connection $db) {
    $db->execute('INSERT INTO audit (event) VALUES (?)', ['import started']);

    try {
        $db->transaction(fn () => importRows($db));   // em caso de falha, volta ao seu savepoint
    } catch (\Throwable $e) {
        logger('Import failed: ' . $e->getMessage());
    }
});   // o registo de auditoria é confirmado em qualquer caso
```

Também é possível controlar as transações manualmente:

| Método | No nível 0 | Quando aninhado |
| --- | --- | --- |
| `beginTransaction()` | `BEGIN` | `SAVEPOINT` |
| `commit()` | `COMMIT` | `RELEASE SAVEPOINT` |
| `rollBack()` | `ROLLBACK` | `ROLLBACK TO SAVEPOINT` |
| `transactionLevel()` | `0` | profundidade de aninhamento |

> **Nota** O MySQL faz commit implícito das instruções DDL (`CREATE TABLE`, `ALTER TABLE`). Não conte com o rollback de alterações ao schema em MySQL.

## Próximos passos

- [Query builder](query-builder.md) — SQL fluente e seguro contra injeção.
- [Modelos](models.md) e [Relações](relationships.md).
- [Migrations e schema](migrations.md) e [Seeders](seeding.md).
