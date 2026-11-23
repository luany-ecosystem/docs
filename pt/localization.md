# Localização

O tradutor do Luany carrega textos a partir de ficheiros PHP em `lang/` e escolhe o idioma ativo em cada pedido. Não tem dependências externas.

## Ficheiros de tradução

Crie um ficheiro por idioma em `lang/`, na raiz do projeto. Cada um devolve um array **plano** — os pontos nas chaves fazem parte do nome da chave, não indicam aninhamento:

```php
// lang/en.php
return [
    'nav.home'         => 'Home',
    'nav.posts'        => 'Posts',
    'welcome.title'    => 'Welcome, :name!',
    'footer.copyright' => '© :year :name',
];
```

```php
// lang/pt.php
return [
    'nav.home'      => 'Início',
    'nav.posts'     => 'Artigos',
    'welcome.title' => 'Bem-vindo, :name!',
];
```

## Traduzir textos

```php
__('nav.home');                                     // 'Home'
__('welcome.title', ['name' => 'Ana']);             // 'Welcome, Ana!'
__('footer.copyright', ['year' => date('Y'), 'name' => 'Luany']);
```

Nas vistas LTE:

```lte
<html lang="{{ locale() }}">
...
<a href="{{ route('home') }}">{{ __('nav.home') }}</a>
```

Ordem de pesquisa: o idioma ativo, depois o idioma de recurso (fallback) e, por fim, a **própria chave** — uma tradução em falta mostra `nav.home` em vez de um texto vazio. Os placeholders escrevem-se `:name` no texto e são passados sem os dois pontos.

O `locale()` devolve o código do idioma ativo.

## Configuração

O tradutor lê o `config/app.php` e recorre às variáveis de ambiente em alternativa:

| Definição | Chave de configuração | Ambiente | Predefinição |
| --- | --- | --- | --- |
| Idioma predefinido | `app.locale` | `APP_LOCALE` | `'en'` |
| Idioma de recurso | `app.fallback_locale` | `APP_FALLBACK_LOCALE` | `'en'` |
| Idiomas suportados | `app.supported_locales` | `APP_SUPPORTED_LOCALES` (separados por vírgulas) | `[locale, fallback]` |

O `config/app.php` do skeleton lista apenas o inglês. Adicione todos os idiomas que disponibilizar:

```php
'locale'            => env('APP_LOCALE', 'en'),
'fallback_locale'   => 'en',
'supported_locales' => ['en', 'pt'],
```

Os idiomas fora de `supported_locales` são rejeitados: o `setLocale()` ignora-os. Inclua também aí o idioma predefinido — se faltar, o tradutor arranca com o idioma de recurso quando este é suportado; caso contrário, com o primeiro idioma suportado.

## Detetar o idioma em cada pedido

Adicione o `LocaleMiddleware` ao middleware global em `app/Http/Kernel.php`:

```php
use Luany\Framework\Http\Middleware\LocaleMiddleware;

protected array $middleware = [
    LiveReloadMiddleware::class,
    CsrfMiddleware::class,
    LocaleMiddleware::class,
];
```

Em cada pedido, escolhe o primeiro idioma suportado de entre:

1. o cookie `app_locale` — a escolha explícita do utilizador;
2. o cabeçalho `Accept-Language` — a preferência do browser, por qualidade (`pt-PT,pt;q=0.9,en;q=0.8` → `pt`; as entradas com `q=0` são ignoradas);
3. o idioma predefinido — `config('app.locale')` e depois a variável de ambiente `APP_LOCALE`, a mesma definição que o tradutor usa;
4. `'en'`, quando é suportado — caso contrário, o tradutor mantém o idioma com que arrancou.

Só é usada a subetiqueta de idioma principal: tanto `pt-BR` como `pt-PT` selecionam `pt`.

## Permitir que os utilizadores mudem de idioma

Defina o cookie `app_locale` e reencaminhe o utilizador para a página anterior. Redirecione apenas para um `Referer` do mesmo host, para evitar um open redirect:

```php
use Luany\Core\Http\Request;
use Luany\Core\Routing\Route;
use Luany\Framework\Support\Url;

Route::get('/locale/{locale}', function (Request $request, string $locale) {
    if (!app('translator')->isSupported($locale)) {
        abort(404);
    }

    $back = Url::sameHostOrFallback($request->header('Referer'), $request->header('Host'));

    return redirect($back)->cookie('app_locale', $locale, time() + 60 * 60 * 24 * 365, secure: $request->isSecure());
})->name('locale');
```

```lte
<a href="{{ route('locale', ['locale' => 'pt']) }}">Português</a>
```

## Locales regionais

Um locale regional recorre à sua língua e depois ao locale de recurso: com `APP_LOCALE=pt_AO`, uma chave é procurada em `lang/pt_AO.php`, depois em `lang/pt.php` e depois em `lang/en.php`. Ponha no ficheiro regional só o que muda.

As mensagens de validação seguem a mesma cadeia. O framework traz-nas em português (`pt`, com o `pt_BR` a trocar "ficheiro" por "arquivo"); qualquer linha pode ser sobreposta no seu `lang/{locale}.php` com a mesma chave, por exemplo `'validation.required' => 'Preencha :field.'`.

## Datas

O `format_date()` escreve uma data no locale atual:

```lte
{{ format_date($post->created_at) }}            {{-- 23/11/2026 --}}
{{ format_date($post->created_at, 'long') }}    {{-- 23 de novembro de 2026 --}}
```

| Estilo | Português | Inglês |
|---|---|---|
| `short` (omissão) | `23/11/2026` | `2026-11-23` |
| `medium` | `23 nov 2026` | `Nov 23, 2026` |
| `long` | `23 de novembro de 2026` | `November 23, 2026` |
| `full` | `segunda-feira, 23 de novembro de 2026` | `Monday, November 23, 2026` |
| `time` | `14:05` | `14:05` |
| `datetime` | `23/11/2026 14:05` | `2026-11-23 14:05` |

As datas são mostradas no fuso da aplicação (`APP_TIMEZONE`). O português e o inglês não precisam da extensão intl; os outros locales usam-na quando está instalada.

## Servidores de longa duração

Com PHP-FPM cada pedido começa do zero. Em servidores de longa duração (Swoole, RoadRunner, workers do FrankenPHP) o tradutor é partilhado entre pedidos: registe o `LocaleMiddleware` para cada pedido definir o seu locale — caso contrário todos os pedidos usam o locale por omissão.

## O tradutor

O tradutor é o `Luany\Framework\Support\Translator`, registado como `translator`:

| Método | Descrição |
| --- | --- |
| `get($key, $replace = [])` | Traduz (é o que o `__()` chama) |
| `setLocale($locale)` | Muda o idioma ativo; os idiomas não suportados são ignorados |
| `getLocale()` | Idioma ativo |
| `getFallback()` | Idioma de recurso |
| `getSupported()` | Idiomas suportados |
| `isSupported($locale)` | Indica se um idioma é permitido |

```php
app('translator')->setLocale('pt');
```

Cada ficheiro de idioma é carregado uma única vez, na primeira utilização. O tradutor não trata a pluralização — use chaves separadas, como `cart.item` e `cart.items`.
