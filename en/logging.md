# Logging

Luany includes a PSR-3 logger that writes to daily files in `storage/logs`. Unhandled exceptions are logged automatically, and any PSR-3 logger (such as Monolog) can replace it.

## Writing log records

Use the `logger()` helper:

```php
logger('Cache warmed');                                   // level: debug
logger('User {id} logged in', ['id' => $user->id]);       // debug, with context
logger('Disk almost full', ['free' => '2GB'], 'warning'); // explicit level

logger()->error('Payment failed', ['order' => 7]);
logger()->info('Import finished');
```

Without arguments, `logger()` returns the `Psr\Log\LoggerInterface`, with the eight PSR-3 methods: `emergency`, `alert`, `critical`, `error`, `warning`, `notice`, `info` and `debug`. An unknown level throws `Psr\Log\InvalidArgumentException`.

The logger is bound as `log` and `Psr\Log\LoggerInterface`, so you can also inject it:

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

## Log format

Each record is one line in `storage/logs/luany-YYYY-MM-DD.log`:

```
[2026-11-10 14:03:22] luany.ERROR: Payment failed {"order":7}
```

- `{placeholders}` in the message are replaced from the context. Dates are written in RFC 3339; arrays and objects become `[array]` / `[object Class]`.
- The remaining context is appended as JSON.
- An exception passed as `$context['exception']` is written with its class, message, file, line and stack trace.
- Line breaks in the message are escaped, so user input cannot forge extra log records.
- If the file cannot be written, the record goes to PHP's `error_log()` instead — logging never breaks a request.

## Configuration

The defaults need no configuration. To change them, create `config/logging.php`:

```php
return [
    'path'    => base_path('storage/logs'),
    'channel' => 'myapp',
    'level'   => env('LOG_LEVEL', 'warning'),
    'daily'   => true,
];
```

| Key | Default | Description |
| --- | --- | --- |
| `path` | `storage/logs` | Directory for log files (created if missing) |
| `channel` | `'luany'` | File name prefix and the name in each line |
| `level` | `LOG_LEVEL`, else `'debug'` | Minimum level written |
| `daily` | `true` | `{channel}-YYYY-MM-DD.log`; `false` writes to `{channel}.log` |

Levels from lowest to highest: `debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`. Records below the minimum are dropped. An invalid level name throws an exception when the logger is created.

A common production setting:

```env
LOG_LEVEL=warning
```

## Exceptions

The [error handler](errors.md#reporting) logs every reported exception at `error` level with the exception in the context. Expected HTTP outcomes listed in the Handler's `$dontReport` (404, 405, `abort()`, validation errors) and `findOrFail()` misses are not logged.

## Using another logger

Bind any PSR-3 logger in a provider's `register()` method; the Kernel then skips its default and `logger()`, injection and exception reporting all use yours:

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

## Using `FileLogger` directly

`Luany\Framework\Log\FileLogger` can also be created for a separate log file:

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
