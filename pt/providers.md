# Service providers

Os service providers são o sítio onde uma aplicação Luany é configurada: registam serviços no [container](container.md) e executam código de configuração depois de estar tudo registado. Esta página aborda o ciclo de vida dos providers, o seu registo e exemplos comuns.

## Escrever um provider

Gere um provider com a CLI:

```bash
luany make:provider Mail
# → app/Providers/MailServiceProvider.php
```

Um provider estende `Luany\Framework\ServiceProvider` e sobrepõe um ou ambos os métodos:

```php
<?php

namespace App\Providers;

use App\Services\Mailer;
use Luany\Framework\Application;
use Luany\Framework\ServiceProvider;

class MailServiceProvider extends ServiceProvider
{
    public function register(Application $app): void
    {
        $app->singleton(Mailer::class, fn () => new Mailer(
            host: (string) env('MAIL_HOST', 'localhost'),
            port: (int) env('MAIL_PORT', 587),
        ));
    }

    public function boot(Application $app): void
    {
        //
    }
}
```

Ambos os métodos estão vazios na classe base, pelo que só implementa o que precisar. Também pode implementar diretamente `Luany\Framework\Contracts\ServiceProviderInterface`, que declara os mesmos dois métodos.

## Ciclo de vida

| Fase | Quando | Para que serve |
|---|---|---|
| `register(Application $app)` | De imediato, quando `$app->register($provider)` é chamado em `bootstrap/app.php` | Registar serviços: `bind()`, `singleton()`, `instance()` |
| `boot(Application $app)` | Uma vez, durante `Kernel::boot()`, depois de **todos** os providers estarem registados e de existirem os serviços predefinidos do framework | Usar serviços: `make()`, registar diretivas, configurar a validação, definir opções estáticas |

A sequência completa de arranque do Kernel HTTP é:

1. Registar os serviços predefinidos do framework (`config`, `session`, `csrf`, `view`, `translator`, `log`) — **apenas os que os seus providers não registaram**.
2. Ligar o container ao router (controllers, middleware, argumentos das ações).
3. Carregar os ficheiros de rotas.
4. Chamar `boot()` em cada provider, pela ordem de registo.
5. Configurar as regras de validação `unique` / `exists`, a menos que um provider já o tenha feito.

> **Aviso** Não chame `make()` em `register()`. Os outros providers — e os serviços predefinidos do framework, como `view` — ainda não estão registados. Resolva os serviços em `boot()`, ou de forma diferida dentro de uma closure de fábrica.

O `boot()` é executado uma vez por aplicação; chamar `bootProviders()` novamente não tem efeito.

## Registar providers

Os providers são registados em `bootstrap/app.php`, depois de o ambiente ser carregado:

```php
// 4. Service Providers — register before kernel boots
$app->register(new App\Providers\AppServiceProvider());
$app->register(new App\Providers\DatabaseServiceProvider());
$app->register(new App\Providers\MailServiceProvider());
```

O `register()` é chamado de imediato, pelo que a ordem só importa quando o `register()` de um provider sobrepõe o registo de outro (vale o último registo). Os métodos `boot()` são executados pela mesma ordem.

Os providers registados podem ser listados com `$app->getProviders()`.

## Substituir serviços do framework

Como o Kernel só regista os seus serviços predefinidos para as chaves que ainda estão livres, um provider pode substituir qualquer um deles em `register()`. Basta registar a chave ou o alias de classe — o Kernel associa o outro ao seu:

```php
use Luany\Lte\Engine;

public function register(Application $app): void
{
    $app->singleton('view', fn (Application $app) => new Engine(
        viewsPath:  $app->basePath('resources/views'),
        cachePath:  $app->cachePath('views'),
        autoReload: (bool) env('APP_DEBUG', false),
    ));
}
```

O mesmo se aplica a `config`, `session`, `csrf`, `translator` e `log` (consulte [Container de serviços](container.md#serviços-do-framework)).

## Exemplo: ligação à base de dados

O `DatabaseServiceProvider` do skeleton mostra a divisão típica. O `register()` apenas declara fábricas — nenhuma ligação é aberta até à primeira query — e o `boot()` entrega a ligação aos modelos:

```php
<?php

namespace App\Providers;

use Luany\Database\Connection;
use Luany\Database\Model;
use Luany\Framework\Application;
use Luany\Framework\ServiceProvider;
use Luany\Framework\Support\Env;

class DatabaseServiceProvider extends ServiceProvider
{
    public function register(Application $app): void
    {
        $app->singleton('connection', function () use ($app) {
            $driver = (string) Env::get('DB_CONNECTION', 'sqlite');
            $name   = (string) Env::get('DB_NAME', $driver === 'sqlite' ? 'database/database.sqlite' : 'luany');

            if ($driver === 'sqlite' && $name !== ':memory:' && !str_starts_with($name, '/') && !preg_match('/^[A-Za-z]:[\\\\\/]/', $name)) {
                $name = $app->basePath($name);
            }

            return Connection::make([
                'driver'   => $driver,
                'host'     => Env::get('DB_HOST', '127.0.0.1'),
                'port'     => Env::get('DB_PORT', $driver === 'pgsql' ? '5432' : '3306'),
                'database' => $name,
                'username' => Env::get('DB_USER', $driver === 'mysql' ? 'root' : null),
                'password' => Env::get('DB_PASS', ''),
            ]);
        });

        // Raw PDO: used by validation rules (unique / exists) and migrations.
        $app->singleton('db', fn (Application $app) => $app->make('connection')->getPdo());
    }

    public function boot(Application $app): void
    {
        Model::setConnection(fn () => $app->make('connection'));
    }
}
```

O binding `db` vai além dos modelos: o Kernel usa-o para alimentar as regras de validação `unique` e `exists`. Consulte [Base de dados: introdução](database.md).

## Exemplo: diretivas LTE próprias

O motor de vistas é um serviço predefinido do framework, pelo que já existe quando o `boot()` é executado. Registe as diretivas no respetivo compilador:

```php
<?php

namespace App\Providers;

use Luany\Framework\Application;
use Luany\Framework\ServiceProvider;

class ViewServiceProvider extends ServiceProvider
{
    public function boot(Application $app): void
    {
        $app->make('view')->getCompiler()->directive('money', function (?string $args): string {
            return "<?php echo \\Luany\\Lte\\Escaper::escape(number_format((float) ({$args}), 2, '.', ',')); ?>";
        });
    }
}
```

```lte
<p>Total: @money($order->total)</p>
```

Os templates compilados ficam em cache por conjunto de diretivas registadas, pelo que acrescentar uma diretiva nunca reutiliza código compilado desatualizado. O [`luany view:check`](view-check.md) arranca a aplicação e, por isso, conhece as diretivas registadas desta forma. Consulte [Diretivas próprias](lte-custom.md).

## Exemplo: presence verifier da validação

As regras `unique` e `exists` perguntam a um *presence verifier* quantas linhas correspondem. Se existir um binding `db`, `connection` ou `PDO`, o Kernel configura automaticamente um `DatabasePresenceVerifier` depois do arranque dos providers. Para usar outra ligação — ou a sua própria pesquisa — defina o verifier em `boot()`; o Kernel não toca num verifier configurado explicitamente:

```php
use Luany\Framework\Validation\DatabasePresenceVerifier;
use Luany\Framework\Validation\Validator;

public function boot(Application $app): void
{
    Validator::setPresenceVerifier(new DatabasePresenceVerifier(
        fn (): \PDO => $app->make('reporting.connection')->getPdo()
    ));
}
```

O `DatabasePresenceVerifier` aceita um `PDO` ou uma closure que devolva um (resolvida na primeira utilização). Para uma fonte que não seja SQL, implemente `Luany\Framework\Validation\PresenceVerifierInterface`:

```php
use Luany\Framework\Validation\PresenceVerifierInterface;

final class ApiPresenceVerifier implements PresenceVerifierInterface
{
    public function count(
        string $table,
        string $column,
        array $values,
        ?string $ignoreColumn = null,
        mixed $ignoreValue = null,
    ): int {
        // Devolve quantos $values distintos existem em $table.$column,
        // excluindo a linha em que $ignoreColumn = $ignoreValue.
        return 0;
    }
}
```

Consulte [Validação](validation.md).

## Exemplo: definições da aplicação

As opções estáticas, válidas para todo o processo, também pertencem ao `boot()` — por exemplo, confiar no seu load balancer:

```php
use Luany\Core\Http\Request;

public function boot(Application $app): void
{
    Request::setTrustedProxies(['10.0.0.0/8']);
}
```
