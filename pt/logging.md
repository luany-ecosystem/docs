# Logging

O Luany inclui um logger PSR-3 que escreve em ficheiros diários em `storage/logs`. As exceções não tratadas são registadas automaticamente, e qualquer logger PSR-3 (como o Monolog) o pode substituir.

## Escrever registos de log

Use o helper `logger()`:

```php
logger('Cache warmed');                                   // nível: debug
logger('User {id} logged in', ['id' => $user->id]);       // debug, com contexto
logger('Disk almost full', ['free' => '2GB'], 'warning'); // nível explícito

logger()->error('Payment failed', ['order' => 7]);
logger()->info('Import finished');
```

Sem argumentos, o `logger()` devolve o `Psr\Log\LoggerInterface`, com os oito métodos PSR-3: `emergency`, `alert`, `critical`, `error`, `warning`, `notice`, `info` e `debug`. Um nível desconhecido lança `Psr\Log\InvalidArgumentException`.

O logger está registado como `log` e como `Psr\Log\LoggerInterface`, pelo que também o pode injetar:

```php
use Psr\Log\LoggerInterface;

class OrderController extends Controller
{
    public function __construct(private LoggerInterface $logger) {}

    public function store(Request $request): Response
    {
        // ...
        $this->logger->info('Order {id} placed', ['id' => $order->id]);
    }
}
```

## Formato do log

Cada registo ocupa uma linha em `storage/logs/luany-YYYY-MM-DD.log`:

```
[2026-11-10 14:03:22] luany.ERROR: Payment failed {"order":7}
```

- Os `{placeholders}` na mensagem são substituídos a partir do contexto. As datas são escritas em RFC 3339; arrays e objetos tornam-se `[array]` / `[object Class]`.
- O restante contexto é acrescentado em JSON.
- Uma exceção passada em `$context['exception']` é escrita com a classe, a mensagem, o ficheiro, a linha e o stack trace.
- As quebras de linha na mensagem são escapadas, para que os dados do utilizador não possam forjar registos de log adicionais.
- Se não for possível escrever no ficheiro, o registo vai para o `error_log()` do PHP — o logging nunca faz falhar um pedido.

## Configuração

As predefinições não precisam de configuração. Para as alterar, crie o `config/logging.php`:

```php
return [
    'path'    => base_path('storage/logs'),
    'channel' => 'myapp',
    'level'   => env('LOG_LEVEL', 'warning'),
    'daily'   => true,
];
```

| Chave | Predefinição | Descrição |
| --- | --- | --- |
| `path` | `storage/logs` | Pasta dos ficheiros de log (criada se não existir) |
| `channel` | `'luany'` | Prefixo do nome do ficheiro e nome em cada linha |
| `level` | `LOG_LEVEL`, ou `'debug'` | Nível mínimo registado |
| `daily` | `true` | `{channel}-YYYY-MM-DD.log`; com `false` escreve em `{channel}.log` |

Níveis, do mais baixo para o mais alto: `debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`. Os registos abaixo do mínimo são descartados. Um nome de nível inválido lança uma exceção quando o logger é criado.

Uma definição comum em produção:

```env
LOG_LEVEL=warning
```

## Exceções

O [handler de erros](errors.md#reporte-de-exceções) regista no log todas as exceções reportadas, ao nível `error`, com a exceção no contexto. Os resultados HTTP esperados listados em `$dontReport` do Handler (404, 405, `abort()`, erros de validação) e as falhas de `findOrFail()` não são registados.

## Usar outro logger

Associe qualquer logger PSR-3 no método `register()` de um provider; o Kernel deixa então de registar o seu logger predefinido, e o `logger()`, a injeção e o reporte de exceções passam todos a usar o seu:

```php
use Monolog\Handler\StreamHandler;
use Monolog\Logger;

public function register(Application $app): void
{
    $app->singleton('log', function (Application $app) {
        $logger = new Logger('app');
        $logger->pushHandler(new StreamHandler($app->storagePath('logs/app.log')));
        return $logger;
    });
}
```

## Usar o `FileLogger` diretamente

O `Luany\Framework\Log\FileLogger` também pode ser criado para um ficheiro de log separado:

```php
use Luany\Framework\Log\FileLogger;

$audit = new FileLogger(
    directory: base_path('storage/logs'),
    channel:   'audit',
    minLevel:  'info',
    daily:     false,
);

$audit->info('Role changed for user {id}', ['id' => 7]);
$audit->path();   // .../storage/logs/audit.log
```
