# Erros de template

O LTE valida cada template ao compilá-lo, antes de executar qualquer PHP. Erros como um `@foreach` por fechar ou uma gralha dentro de `{{ }}` são reportados com o nome da vista, a linha e a coluna. Esta página explica o que é verificado, o aspeto dos erros, como os erros de execução são associados às linhas do `.lte` e como funciona a cache de compilação.

## O pipeline de compilação

```
.lte source → Parser (tokens with line/column)
            → Ast\TreeBuilder (syntax tree + validation)
            → Compiler (PHP) → cache file → evaluate → HTML
```

O parser foi escrito à mão (sem expressões regulares) e tem em conta strings e parênteses, por isso `{{ '}}' }}` e `@include('card', ['a' => 1, 'b' => 2])` são interpretados corretamente.

## O que é validado

| Verificação | Exemplo de erro |
| --- | --- |
| Todos os blocos estão fechados | `Unclosed @foreach: missing @endforeach (line 12, column 5).` |
| Cada fecho corresponde ao bloco mais interior | `Unexpected @endif: @foreach opened on line 3 is still open (expected @endforeach) (line 4, column 1).` |
| Não há fechos soltos | `Unexpected @endsection: there is no open block to close (line 9, column 1).` |
| Os ramos estão no bloco certo | `Unexpected @else inside @foreach (line 12) (line 15, column 3).` |
| `@else` é o último ramo | `Unexpected @elseif: @else (line 6) must be the last branch of @if (line 8, column 1).` |
| `@empty` sem argumentos só uma vez, dentro de `@forelse` | `Duplicate @empty in @forelse (line 10, column 1).` |
| Os argumentos obrigatórios estão presentes | `@include requires arguments, e.g. @include(…) (line 3, column 1).` |
| O PHP embutido é sintaticamente válido | `PHP syntax error in {{ }} on line 2, column 4: syntax error, unexpected token ")".` |

O PHP é verificado em `{{ }}`, `{!! !!}`, blocos `@php` e argumentos de diretivas com o próprio parser do PHP (`token_get_all()` com `TOKEN_PARSE`). Nada é executado. As tags por fechar (`{{` sem `}}`, `{{--` sem `--}}`, `@php` sem `@endphp`) são reportadas pelo parser.

As [diretivas próprias](lte-custom.md) não são validadas: o LTE não conhece os seus argumentos nem a sua estrutura.

Quando o motor compila uma vista, a mensagem é prefixada com o nome da vista:

```
LTE compilation error in view [pages.users]: Unclosed @foreach: missing @endforeach (line 12, column 5).
```

### Fechos colados

Uma diretiva de fecho tem de estar separada do texto que se lhe segue. `Title@endslotBody` é uma única palavra desconhecida (`endslotBody`), por isso fica como texto e o slot continua aberto. O LTE deteta esta situação e acrescenta uma sugestão:

```lte
@component('components.card')
    @slot('title')Title@endslotBody
@endcomponent
```

```
Unexpected @endcomponent: @slot opened on line 2 is still open (expected @endslot)
— "@endslotBody" on line 2 is read as text: separate the directive from the text ("@endslot Body") (line 3, column 1).
```

Texto imediatamente *antes* de um fecho não causa problemas: `Title@endslot` funciona.

## `TemplateSyntaxException`

Os erros de compilação lançam `Luany\Lte\TemplateSyntaxException`, que estende `\RuntimeException` (os blocos `catch (\RuntimeException)` existentes continuam a funcionar). Expõe:

| Propriedade | Tipo | Descrição |
| --- | --- | --- |
| `$view` | `?string` | Nome da vista em notação de pontos, quando conhecido. |
| `$templateLine` | `int` | Linha no ficheiro `.lte`, a contar de 1 (`0` = desconhecida). |
| `$templateColumn` | `int` | Coluna a contar de 1, em bytes (`0` = desconhecida). |

```php
use Luany\Lte\TemplateSyntaxException;

try {
    $html = view('pages.report', $data);
} catch (TemplateSyntaxException $e) {
    logger('Broken template', [
        'view'   => $e->view,
        'line'   => $e->templateLine,
        'column' => $e->templateColumn,
    ], 'error');
    throw $e;
}
```

A mesma exceção é lançada em tempo de renderização para problemas estruturais que o compilador não consegue ver, como uma cadeia circular de `@extends` ou uma vista que deixa um buffer de output aberto.

## Verificar todas as vistas

Execute o validador sobre toda a pasta `views/` sem renderizar nada — ideal para CI:

```bash
luany view:check
```

Ver [view:check](view-check.md) para as opções e as anotações do GitHub Actions.

### Inspecionar a árvore sintática

As ferramentas podem analisar e validar uma vista com `Engine::parseView()`. Este método lança `TemplateSyntaxException` (com a mensagem prefixada por `In view [name]:`) ou `\RuntimeException` quando a vista não existe; caso contrário, devolve um `Ast\Template`:

```php
use Luany\Lte\Ast\BlockNode;
use Luany\Lte\Ast\Node;

$tree = app('view')->parseView('pages.home');

$tree->walk(function (Node $node, int $depth): void {
    if ($node instanceof BlockNode) {
        echo str_repeat('  ', $depth) . "@{$node->name} (line {$node->line})\n";
    }
});
```

| Nó | Descrição |
| --- | --- |
| `Ast\Template` | Raiz. `$children`, `walk(callable $visitor)` (em profundidade, cada bloco antes dos seus filhos), `tokens()`. |
| `Ast\BlockNode` | Uma diretiva de bloco: `$name`, `$branches` (lista de `Ast\Branch`), `$closing`, `opening()`. |
| `Ast\Branch` | Um ramo (`@if`, `@elseif`, `@else`, `@empty`, …): `$directive` e `$children`. |
| `Ast\TokenNode` | Folha: texto, echo, echo sem escape, bloco PHP ou diretiva que não é de bloco. `type()`, `name()`, `$token`. |

Todos os nós têm `$line` e `$column`. A API do compilador usada pelo motor também é pública: `Compiler::buildTree(array $tokens)`, `Compiler::compileTree(Template $tree)` e `Compiler::validate(array $tokens)`.

## Erros em tempo de execução

As exceções lançadas durante a execução de uma vista compilada (um método inexistente, um `TypeError`) são relançadas como `\RuntimeException` com o ficheiro `.lte` e a linha originais; a exceção original está disponível através de `getPrevious()`:

```
LTE render error in [home.lte line 23]: Call to undefined method App\Models\User::missing()
```

O compilador insere marcadores de linha `@lte:N` no PHP gerado, e o motor faz corresponder a linha de PHP que falhou ao marcador mais próximo. Se uma renderização falhar a meio, os buffers de output abertos e o estado das secções, stacks e componentes são desfeitos, para que os workers de longa duração comecem o pedido seguinte num estado limpo.

Em desenvolvimento, a [página de erro](errors.md) do framework mostra estas mensagens; em produção, são registadas no log.

## A cache de compilação

Os templates compilados são ficheiros PHP simples com o nome dado por um hash SHA-256.

| Aspeto | Comportamento |
| --- | --- |
| Localização | `storage/cache/views` numa aplicação Luany. Um `Engine` autónomo sem `$cachePath` usa uma pasta privada por utilizador (`sys_get_temp_dir()/lte_cache_<uid>`, modo `0700`) e recusa uma pasta que seja um symlink ou pertença a outro utilizador. |
| Chave da cache | Versão do motor (`Engine::VERSION`, a versão do LTE instalada) + nomes das diretivas próprias registadas + caminho da vista. Atualizar o LTE ou registar uma diretiva nunca reaproveita código desatualizado. |
| Atualidade | Com `autoReload` desativado, uma vista é recompilada quando o código-fonte é pelo menos tão recente como o ficheiro compilado (as edições no mesmo segundo são detetadas). Com `autoReload` ativo (`APP_DEBUG=true` no framework), todas as renderizações recompilam. |
| Escritas atómicas | Cada compilação escreve um ficheiro temporário e move-o para o lugar com `rename()`, para que pedidos concorrentes nunca leiam um ficheiro escrito a meio. A OPcache é invalidada para o novo ficheiro. |

Limpe a cache depois de cada deploy, ou depois de alterar o handler de uma diretiva própria:

```bash
luany cache:clear
```

`Engine::clearCache()` faz o mesmo a partir de PHP.
