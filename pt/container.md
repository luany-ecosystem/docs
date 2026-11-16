# Container de serviços

`Luany\Framework\Application` é o container de injeção de dependências do framework. Regista serviços, constrói classes com as respetivas dependências através de autowiring, injeta argumentos nas ações dos controllers e implementa a PSR-11.

## Aceder ao container

A instância da aplicação é criada em `bootstrap/app.php` e está disponível em qualquer lado através do helper `app()`:

```php
$app = app();                 // a Application
$view = app('view');          // o mesmo que app()->make('view')
```

Dentro de um service provider, o container é passado a `register()` e `boot()` como `$app`. A Application também está registada em si mesma, pelo que um construtor que declare o tipo `Luany\Framework\Application` ou `Psr\Container\ContainerInterface` a recebe.

## Registar serviços

| Método | Tempo de vida | Descrição |
|---|---|---|
| `bind($abstract, $concrete = null)` | Transitório | Uma nova instância a cada `make()` |
| `singleton($abstract, $concrete = null)` | Partilhado | Construído no primeiro `make()` e depois guardado em cache |
| `instance($abstract, $object)` | Partilhado | Regista um objeto que já construiu |

`$abstract` é qualquer string: um nome de classe ou de interface, ou uma chave curta como `'mailer'`. `$concrete` pode ser:

- um **callable de fábrica** — recebe a Application e um array de parâmetros;
- um **nome de classe** — construído por autowiring (normalmente para associar uma interface a uma implementação);
- **null** — faz o autowiring do próprio `$abstract`.

```php
use App\Services\Mailer;
use App\Services\SmtpTransport;
use App\Contracts\TransportInterface;
use Luany\Framework\Application;

// Fábrica
$app->singleton('mailer', fn (Application $app) => new Mailer(
    host: (string) env('MAIL_HOST', 'localhost'),
    port: (int) env('MAIL_PORT', 587),
));

// Interface → implementação
$app->bind(TransportInterface::class, SmtpTransport::class);

// Partilhado e com autowiring
$app->singleton(Mailer::class);

// Objeto já construído
$app->instance('clock', new \DateTimeImmutable('2026-01-01'));
```

Chamar `singleton()` novamente para a mesma chave substitui o registo e descarta uma instância construída anteriormente.

## Resolver serviços

```php
$mailer = $app->make('mailer');
$report = $app->make(ReportService::class, ['year' => 2026]);
```

O `make()` procura uma instância registada, depois um singleton e depois um binding transitório. Se não houver nada registado e o identificador for uma classe existente, a classe é construída por autowiring. O segundo argumento, opcional, passa **argumentos nomeados do construtor**, que se sobrepõem à resolução automática.

| Exceção | Quando |
|---|---|
| `Luany\Framework\Container\NotFoundException` | Não há nada registado e a classe não existe |
| `Luany\Framework\Container\ContainerException` | A entrada existe, mas não pode ser construída (classe abstrata, parâmetro impossível de resolver, binding inválido) |
| `Luany\Framework\Container\CircularDependencyException` | A entrada depende de si própria |

As três estendem `ContainerException`, que estende `\RuntimeException` e implementa as interfaces de exceção da PSR-11.

## Autowiring

As classes concretas sem registo são construídas por reflexão. Cada parâmetro do construtor é resolvido por esta ordem:

1. Um argumento nomeado passado ao `make()`.
2. O seu tipo de classe ou interface: `Application` / `ContainerInterface` recebem o container; qualquer outro tipo é resolvido com `make()`. As interfaces têm de estar registadas; caso contrário, passa-se à regra seguinte.
3. O seu valor predefinido.
4. `null`, quando o parâmetro aceita null.

Caso contrário, é lançada uma `ContainerException` que indica o parâmetro e a classe.

```php
namespace App\Services;

use Psr\Log\LoggerInterface;

class ReportService
{
    public function __construct(
        private LoggerInterface $log,   // o logger do framework
        private Mailer $mailer,         // autowiring (ou o singleton registado)
        private int $year = 2026,       // valor predefinido, ou make(..., ['year' => 2025])
    ) {}
}
```

```php
$report = app(ReportService::class);
```

Os controllers, os middlewares indicados por nome de classe e os middlewares globais são todos criados através do container, pelo que as mesmas regras lhes dão injeção no construtor:

```php
class InvoiceController extends Controller
{
    public function __construct(private ReportService $reports) {}
}
```

### Dependências circulares

Se `A` precisa de `B` e `B` precisa de `A`, a resolução para com uma `CircularDependencyException` que mostra a cadeia completa, em vez de entrar em recursão infinita:

```text
Circular dependency detected while resolving [App\A]: App\A -> App\B -> App\A
```

## Injeção em métodos com call()

O `call()` invoca uma closure, um objeto invocável, `[object, 'method']` ou `[ClassName::class, 'method']` (a classe é resolvida através do container), resolvendo cada parâmetro:

1. **Por nome** — uma chave string em `$parameters` igual ao nome do parâmetro.
2. **Por tipo** — um valor em `$parameters` que seja instância da classe declarada; caso contrário, o container constrói-a.
3. **Por posição** — os restantes valores de `$parameters`, pela ordem.
4. **Valor predefinido** e, depois, `null` para parâmetros que aceitem null.

```php
$total = $app->call(
    [InvoiceService::class, 'total'],
    ['currency' => 'AOA'],
);

class InvoiceService
{
    public function total(ReportService $reports, string $currency, int $year = 2026): string
    {
        // $reports vem do container, $currency pelo nome, $year pelo valor predefinido
        return number_format($reports->totalFor($year), 2) . ' ' . $currency;
    }
}
```

Um parâmetro obrigatório que nenhuma destas regras consiga preencher lança uma `ContainerException`.

### Ações dos controllers

O Kernel HTTP chama todas as ações de rota através de `call()`, passando o pedido e os parâmetros da rota pelo nome. É por isso que todas estas assinaturas funcionam:

```php
Route::get('/posts/{id}', [PostController::class, 'show']);

public function show(Request $request, int $id) { /* ... */ }
public function show(int $id, PostRepository $posts) { /* ... */ }  // a ordem não importa
public function show($request, $slug) { /* posicional: o pedido e depois o id */ }
```

Quando um route-model binding resolve para `null` num parâmetro que exige um objeto (por exemplo, `show(Post $post)` sem nenhum post encontrado), o Kernel responde com **404**. Declare o parâmetro como nullable (`?Post $post`) para tratar o caso em falta por si. Consulte [Rotas](routing.md) e [Controllers](controllers.md).

## PSR-11

A Application implementa `Psr\Container\ContainerInterface`:

| Método | Comportamento |
|---|---|
| `get(string $id)` | O mesmo que `make($id)` |
| `has(string $id)` | `true` apenas para entradas **registadas explicitamente** (`bind`, `singleton`, `instance`) |
| `bound(string $id)` | Alias de `has()` |

O `has()` devolve `false` para classes que admitem autowiring mas nunca foram registadas — o `get()` consegue construí-las na mesma. Tenha isso em conta quando um consumidor PSR-11 verifica `has()` antes de `get()`.

```php
use Psr\Container\ContainerInterface;

function handler(ContainerInterface $container): object
{
    return $container->get(ReportService::class);
}

handler(app());
```

Consulte [Interoperabilidade PSR](psr.md) para usar o container com middleware PSR-15 e com outros containers PSR-11.

## Serviços do framework

Durante o arranque, o Kernel HTTP regista estes serviços predefinidos, a menos que um service provider já os tenha registado, cada um sob uma chave e um alias de classe:

| Chave | Alias de classe | Serviço |
|---|---|---|
| `config` | `Luany\Framework\Support\Config` | Configuração a partir de `config/*.php` |
| `session` | `Luany\Framework\Contracts\SessionInterface` | Sessão em ficheiros |
| `csrf` | `Luany\Framework\Security\CsrfToken` | Token CSRF |
| `view` | `Luany\Lte\Engine` | Motor LTE |
| `translator` | `Luany\Framework\Support\Translator` | Traduções a partir de `lang/` |
| `log` | `Psr\Log\LoggerInterface` | Logger diário em ficheiro |

Para substituir um deles, registe a chave ou a classe no `register()` de um provider — consulte [Service providers](providers.md).
