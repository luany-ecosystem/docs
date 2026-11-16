# view:check

`luany view:check` valida todas as vistas LTE de um projeto sem renderizar nada, e termina com um código diferente de zero quando alguma vista está partida. Use-o localmente, no editor e em CI.

## Utilização

```bash
luany view:check
```

```bash
  →  Checking 9 view(s) in views/

  ✗  views/pages/products/index.lte:42:21
     Unclosed @foreach: missing @endforeach.

  ✗  1 of 9 view(s) have errors.
```

Quando está tudo válido:

```bash
  →  Checking 9 view(s) in views/

  ✓  9 view(s) OK.
```

O comando exige `luany/lte` 1.2 ou mais recente no projeto (caso contrário, `composer update luany/lte`) e luany/cli 1.4 ou mais recente.

## O que é verificado

Cada ficheiro `.lte` dentro da pasta de vistas (recursivamente, por ordem alfabética) é analisado e validado — nada é executado e não é escrito nenhum ficheiro compilado:

- cada bloco é fechado pela diretiva certa; não há fechos soltos nem cruzados (`@if` … `@endforeach`);
- `@elseif` / `@else` só dentro de blocos do tipo `@if`, e um `@empty` simples apenas uma vez dentro de `@forelse`;
- as diretivas que precisam de argumentos têm-nos (`@include`, `@section`, …);
- o PHP dentro de `{{ }}`, `{!! !!}`, `@php` e dos argumentos das diretivas tem sintaxe válida.

Os erros são indicados no formato `file:line:column`, com a mensagem mais interior. Os problemas de execução — uma variável indefinida, uma vista incluída inexistente — não são detetados, porque nada é renderizado. Consulte [Erros de template](lte-errors.md) para a lista completa das mensagens de compilação.

## Opções

| Opção | Predefinição | Descrição |
|---|---|---|
| `--path=<dir>` | `views` | Pasta das vistas, relativa à raiz do projeto. `..` não é permitido. |
| `--format=text\|github` | `text` | `github` produz anotações do GitHub Actions em vez do relatório legível |
| `--no-app` | desativada | Não arranca a aplicação; usa um motor LTE simples |

As opções desconhecidas são rejeitadas com uma mensagem de utilização.

```bash
luany view:check --no-app
luany view:check --path=resources/views --no-app
luany view:check --format=github
```

## Arranque da aplicação

Por omissão, o `view:check` arranca a sua aplicação — carrega `bootstrap/app.php`, arranca o Kernel HTTP e usa o motor `view` registado no container. Assim, as **diretivas próprias registadas pelos seus service providers** são conhecidas e validadas como as nativas. Todo o output produzido durante o arranque é descartado.

Se o arranque falhar (por exemplo, se faltar o `.env` e `Env::required()` lançar uma exceção), o comando recorre a um motor simples e informa-o:

```bash
  ⚠  Could not boot the application (Required environment variables are not set: APP_ENV, APP_URL).
     Checking without directives registered by your providers.
```

Nesse modo, e com `--no-app`, qualquer `@name` que não seja uma diretiva nativa é tratado como texto simples, pelo que uma vista que use diretivas próprias continua a ser verificada quanto à estrutura e à sintaxe PHP.

> **Aviso** Quando a aplicação arranca, os nomes das vistas são resolvidos pelo motor da aplicação, que lê da pasta `views/` da aplicação — os ficheiros encontrados noutro `--path` são então reportados como "not found". Combine sempre `--path` com `--no-app`.

O `--no-app` é mais rápido e não tem efeitos secundários; é o que a extensão do VS Code usa por omissão.

## Códigos de saída

| Código | Significado |
|---|---|
| `0` | Todas as vistas são válidas |
| `1` | Pelo menos uma vista tem erros, a pasta de vistas não existe, `luany/lte` é anterior à 1.2, uma opção é inválida, ou o comando foi executado fora de um projeto Luany |

## Anotações do GitHub

Com `--format=github`, as linhas de resumo são suprimidas e cada erro torna-se um workflow command:

```bash
::error file=views/pages/products/index.lte,line=42,col=21::Unclosed @foreach: missing @endforeach.
```

O GitHub Actions transforma-os em anotações nas linhas alteradas do pull request. O output é também estável para outras ferramentas — a [extensão do VS Code](vscode.md) analisa-o para sublinhar os erros no editor.

## Exemplo de CI

O skeleton não depende de `luany/cli`, por isso instale-a no job. Este workflow do GitHub Actions verifica as vistas em cada push e pull request:

```yaml
name: Views

on: [push, pull_request]

jobs:
  view-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: shivammathur/setup-php@v2
        with:
          php-version: "8.2"
          extensions: pdo, pdo_sqlite, mbstring

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist

      - name: Install the Luany CLI
        run: composer global require luany/cli --no-interaction

      - name: Prepare .env
        run: cp .env.example .env

      - name: Check LTE views
        run: $(composer global config bin-dir --absolute)/luany view:check --format=github
```

O passo do `.env` permite que a aplicação arranque, para que as diretivas registadas pelos providers sejam validadas. Se os seus providers precisarem de serviços que não estão disponíveis em CI (um servidor de base de dados, chaves de API), retire esse passo e acrescente `--no-app`.

Em alternativa, adicione a CLI como dependência de desenvolvimento e execute `vendor/bin/luany view:check --format=github`:

```bash
composer require --dev luany/cli
```

## Utilização programática

O comando assenta no método `parseView()` do motor, que pode chamar a partir das suas próprias ferramentas:

```php
use Luany\Lte\Engine;
use Luany\Lte\TemplateSyntaxException;

$engine = new Engine(__DIR__ . '/views', sys_get_temp_dir() . '/lte-check');

try {
    $engine->parseView('pages.home');
} catch (TemplateSyntaxException $e) {
    echo "{$e->view}:{$e->templateLine}:{$e->templateColumn} {$e->getMessage()}\n";
}
```
