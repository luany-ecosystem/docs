# Configuração

O Luany lê definições de dois sítios: variáveis de ambiente (normalmente um ficheiro `.env`) para valores que mudam de máquina para máquina, e ficheiros PHP em `config/` para as definições da aplicação.

## Variáveis de ambiente

Um projeto novo inclui um `.env.example`; o `luany new` e o `composer create-project` copiam-no para `.env`:

```env
APP_NAME=Luany
# development | production — production desativa as páginas de debug e o live reload.
APP_ENV=development
APP_DEBUG=true
APP_URL=http://localhost:8000
APP_KEY=
APP_TIMEZONE=UTC
APP_LOCALE=en

# sqlite (não precisa de servidor) | mysql | pgsql
DB_CONNECTION=sqlite
DB_NAME=database/database.sqlite
```

| Variável | Finalidade |
| --- | --- |
| `APP_NAME` | Nome da aplicação (`config('app.name')`) |
| `APP_ENV` | `development` ou `production`. O live reload só funciona em `development` (ou `local`) |
| `APP_DEBUG` | `true` mostra a página de erro de debug e recompila as vistas alteradas; sempre `false` em produção |
| `APP_URL` | URL base usado por `url()`, `route()` e `asset()` |
| `APP_KEY` | Chave aleatória gerada pelo instalador e por `luany key:generate` |
| `APP_TIMEZONE` | Fuso horário predefinido do PHP |
| `APP_LOCALE` | Idioma predefinido para as [traduções](localization.md) |
| `LOG_LEVEL` | Nível mínimo de [log](logging.md) (predefinido: `debug`) |
| `DB_CONNECTION`, `DB_NAME`, `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS` | Ligação à base de dados — consulte [Base de dados](database.md) |

> **Aviso** O `.env` contém segredos e está listado no `.gitignore`. Faça antes commit do `.env.example`, com valores vazios ou fictícios.

### Como o `.env` é carregado

O `bootstrap/app.php` carrega o ficheiro e verifica as variáveis sem as quais a aplicação não pode funcionar:

```php
Env::load($ROOT);
Env::required(['APP_ENV', 'APP_URL']);
```

- `Env::load()` não faz nada se o `.env` não existir, e só é executado uma vez por processo.
- As variáveis já definidas no ambiente real (configuração do servidor, Docker, CI) **têm prioridade sobre o `.env`**: o ficheiro nunca as sobrepõe. Em produção pode dispensar o `.env` por completo e definir variáveis de ambiente reais.
- `Env::required()` lança uma `RuntimeException` com a lista de todas as variáveis em falta ou vazias.

### Ler valores

Use o helper `env()` (ou `Env::get()`), que consulta `$_ENV`, depois `$_SERVER` e depois `getenv()`:

```php
env('APP_NAME', 'Luany');   // 'Luany' quando não está definida
env('DB_PORT', 3306);
```

Alguns valores de texto são convertidos automaticamente:

| Valor no `.env` | `env()` devolve |
| --- | --- |
| `true`, `(true)` | `true` |
| `false`, `(false)` | `false` |
| `null`, `(null)` | `null` |
| `empty`, `(empty)` | `''` |

Tudo o resto é devolvido como string, por isso faça o cast quando necessário: `(int) env('DB_PORT', 3306)`.

> **Dica** Chame `env()` apenas a partir dos ficheiros de configuração e dos service providers. No resto do código, leia o valor de configuração (`config('app.debug')`), para que haja um único sítio a fazer a correspondência entre o ambiente e as definições.

## Ficheiros de configuração

Cada ficheiro PHP em `config/` que devolve um array torna-se uma chave de primeiro nível com o nome do ficheiro. O skeleton inclui o `config/app.php`:

```php
return [
    'name'     => env('APP_NAME', 'Luany'),
    'env'      => env('APP_ENV', 'production'),
    'debug'    => (bool) env('APP_DEBUG', false),
    'url'      => env('APP_URL', 'http://localhost:8000'),
    'timezone' => env('APP_TIMEZONE', 'UTC'),

    // As traduções ficam em lang/{locale}.php — use __('key') no código e nas vistas.
    'locale'            => env('APP_LOCALE', 'en'),
    'fallback_locale'   => 'en',
    'supported_locales' => ['en'],
];
```

| Chave | Usada para |
| --- | --- |
| `app.name` | Títulos das páginas no layout do skeleton |
| `app.env` | As suas próprias verificações de ambiente |
| `app.debug` | Modo de debug das diretivas `@dump` / `@dd` do LTE |
| `app.url` | `url()`, `route()`, `asset()` (recorre a `APP_URL` em alternativa) |
| `app.timezone` | `date_default_timezone_set()` no arranque (nomes inválidos são ignorados) |
| `app.locale`, `app.fallback_locale`, `app.supported_locales` | O [tradutor](localization.md) |

O framework lê também estes ficheiros opcionais, quando existem:

| Ficheiro | Página |
| --- | --- |
| `config/session.php` | [Sessão e mensagens flash](session.md#configuração) |
| `config/logging.php` | [Logging](logging.md#configuração) |

Pode adicionar os seus próprios ficheiros à vontade — `config/services.php` fica disponível como `config('services.…')`.

## Aceder à configuração

O helper `config()` lê valores aninhados com notação de pontos:

```php
config('app.name');                  // 'Luany'
config('app.missing', 'default');    // 'default'
config('services.stripe.key');

config(['app.debug' => false]);      // define um ou mais valores em tempo de execução
config()->has('app.timezone');       // true
config()->all();                     // todos os valores carregados
```

As alterações em tempo de execução só duram até ao fim do pedido atual; não são escritas de volta no ficheiro.

O repositório é a classe `Luany\Framework\Support\Config`, registada no container como `config`, pelo que também a pode injetar:

```php
use Luany\Framework\Support\Config;

public function index(Config $config): string
{
    return $config->get('app.name');
}
```

## Modo de debug

O `APP_DEBUG` é lido em mais do que um sítio:

- O `bootstrap/app.php` passa-o ao [handler de erros](errors.md), que mostra a página de debug detalhada para erros de servidor.
- O motor de vistas recompila os templates quando o código-fonte muda.
- `config('app.debug')` ativa as diretivas `@dump` e `@dd` do LTE.

> **Aviso** Nunca defina `APP_DEBUG=true` num servidor público: a página de debug mostra caminhos de ficheiros, a mensagem da exceção e o stack trace completo.
