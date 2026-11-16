# Extensão do VS Code

A extensão oficial **Luany LTE** para o Visual Studio Code acrescenta realce de sintaxe, snippets, um ícone de ficheiro e diagnóstico de erros em tempo real para os templates `.lte`. Esta página aborda a instalação, as funcionalidades e as definições.

## Instalação

### A partir do Marketplace

1. Abra a vista de extensões (`Ctrl+Shift+X`, ou `Cmd+Shift+X` no macOS).
2. Procure **Luany LTE** (editor `luany`).
3. Clique em **Install**.

Ou a partir do terminal:

```bash
code --install-extension luany.luany-lte
```

### A partir de um ficheiro VSIX

Descarregue o pacote `.vsix` das releases da extensão no GitHub e depois:

```bash
code --install-extension luany-lte-1.2.0.vsix
```

ou use **Extensions → … → Install from VSIX…** no editor. A extensão requer o VS Code 1.80 ou mais recente e é ativada para ficheiros `.lte` e para workspaces que os contenham.

## Realce de sintaxe

Os ficheiros `.lte` têm um modo de linguagem próprio (**Luany LTE**) com:

- realce de todas as diretivas LTE, com argumentos PHP em qualquer nível de parênteses;
- PHP embutido em `{{ }}`, `{!! !!}`, blocos `@php` e argumentos das diretivas;
- `{{ }}` e `{!! !!}` reconhecidos em texto, valores de atributos HTML, `<script>` e `<style>`;
- CSS embutido em blocos `@style` e JavaScript em blocos `@script`;
- os escapes `@@` e `@{{ }}` mostrados como literais; endereços de email e outro texto do tipo `word@word` nunca são confundidos com diretivas.

A extensão configura também a indentação automática, o fecho automático de pares para as construções LTE e as abreviaturas HTML do Emmet nos ficheiros `.lte`. As suas próprias definições do editor, como a quebra de linha e a formatação ao escrever, ficam intactas.

## Snippets

Escreva um prefixo e prima `Tab`. Os prefixos entre parênteses são aliases.

| Prefixo | Insere |
|---|---|
| `layout` | Estrutura oficial de uma vista LTE |
| `component` | Componente autónomo com CSS e JS no mesmo ficheiro |
| `component-use` (`@component`) | Componente com um slot nomeado |
| `@slot` (`slot`) | Slot nomeado dentro de um componente |
| `if` | `@if` / `@endif` |
| `ifelse` | `@if` / `@else` / `@endif` |
| `unless` (`@unless`) | `@unless` / `@endunless` |
| `isset` (`@isset`) | `@isset` / `@endisset` |
| `ifempty` (`@ifempty`) | `@ifempty` / `@endifempty` |
| `foreach` | `@foreach` / `@endforeach` |
| `forelse` | `@forelse` / `@empty` / `@endforelse` com um estado vazio |
| `for` (`@for`) | `@for` / `@endfor` |
| `while` (`@while`) | `@while` / `@endwhile` |
| `section` | `@section` / `@endsection` |
| `style` | Bloco `@style` (recolhido pela stack de assets) |
| `script` | Bloco `@script(defer)` (recolhido pela stack de assets) |
| `push` | `@push` / `@endpush` |
| `include` | `@include` de um partial |
| `include-data` | `@include` de um partial com dados |
| `php` (`@php`) | `@php` / `@endphp` |
| `@json` (`json`) | Output JSON seguro para JavaScript inline |
| `@class` (`class`) | Classes CSS condicionais |
| `@method` (`method`) | Campo para simular o método HTTP |
| `form` | Formulário `POST` com `@csrf` |
| `form-method` | Formulário com `@csrf` e `@method` |
| `csrf` | Campo oculto CSRF |
| `auth` | `@auth` / `@endauth` |
| `guest` | `@guest` / `@endguest` |
| `dump` (`@dump`) | `@dump($var)` — não faz nada com o debug desligado |
| `dd` (`@dd`) | `@dd($var)` — não faz nada com o debug desligado |
| `@{{` | `{{ }}` literal (não compilado) |

Consulte a [Referência de diretivas](lte-directives.md) para saber o que faz cada diretiva.

## Ícone de ficheiro

Os ficheiros `.lte` mostram o ícone do Luany no Explorer, nos separadores do editor e no Quick Open.

O VS Code só usa o ícone próprio de uma linguagem quando o **File Icon Theme** ativo não tem um ícone específico para esse tipo de ficheiro. É o caso dos temas nativos **Seti** e **Minimal**. Os temas que incluem um conjunto completo de ícones, como o *Material Icon Theme*, podem continuar a mostrar o seu ícone genérico para os ficheiros `.lte` — mude de tema (**Preferences: File Icon Theme**) ou configure as associações de ficheiros desse tema se quiser o ícone do Luany.

## Diagnóstico de erros

Quando abre ou guarda um ficheiro `.lte`, a extensão executa [`luany view:check`](view-check.md) no projeto e sublinha cada problema na linha e coluna exatas. Os mesmos problemas aparecem no painel **Problems** (origem `luany view:check`):

- blocos por fechar, soltos ou cruzados (`@if` … `@endforeach`);
- `@else`, `@elseif` ou `@empty` fora do sítio, e diretivas sem os argumentos necessários;
- erros de sintaxe PHP dentro de `{{ }}`, `{!! !!}`, `@php` e argumentos das diretivas.

O diagnóstico abrange todo o projeto, não apenas o ficheiro aberto, e é substituído em cada execução. As execuções passam por um debounce (300 ms) e nunca se sobrepõem para o mesmo projeto.

Para validar quando quiser, abra a Command Palette e execute **Luany LTE: Validate views**. O resultado aparece na barra de estado: `all views OK` ou o número de problemas.

### Requisitos

- luany/cli 1.4 ou mais recente, instalada globalmente (`composer global require luany/cli`) ou no projeto (`composer require --dev luany/cli`).
- `luany/lte` 1.2 ou mais recente no projeto.

A raiz do projeto é a pasta mais próxima acima do ficheiro que contenha `bootstrap/app.php`, ou simultaneamente `views/` e `composer.json`. A CLI é procurada por esta ordem:

1. A definição `luanyLte.cliPath`.
2. `vendor/bin/luany` no projeto (executado com `php`).
3. `luany` no seu `PATH`.

Se não for encontrada nenhuma CLI, a extensão mostra um aviso, uma única vez, com o comando de instalação. As outras falhas são escritas no canal de output **Luany LTE**.

### Definições

| Definição | Predefinição | Descrição |
|---|---|---|
| `luanyLte.validate.enabled` | `true` | Valida ao abrir e ao guardar. Desativá-la limpa o diagnóstico. |
| `luanyLte.validate.bootApp` | `false` | Arranca a aplicação durante a validação, para que as diretivas registadas pelos seus service providers sejam conhecidas. Mais lento. |
| `luanyLte.cliPath` | `""` | Caminho para o executável `luany`. Vazio: `vendor/bin/luany` e depois `luany` no `PATH`. |

```json
{
    "luanyLte.validate.enabled": true,
    "luanyLte.validate.bootApp": true,
    "luanyLte.cliPath": "/home/me/.config/composer/vendor/bin/luany"
}
```

Com o `bootApp` desligado (o predefinido), a extensão passa `--no-app`: as diretivas próprias são tratadas como texto simples, mas a estrutura e a sintaxe PHP continuam a ser verificadas. Ative-o se os seus providers registarem diretivas e o seu `.env` permitir que a aplicação arranque.
