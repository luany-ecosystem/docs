# Programming in Portuguese

Luany can be written in Portuguese end to end — views, framework API, validation rules and CLI messages — while running plain PHP underneath. Every layer is opt-in, English keeps working, and both can be mixed.

| Layer | How to turn it on | Since |
|---|---|---|
| Views: `@se`, `@paracada`, `@secao`… | `LUANY_SYNTAX=pt` | luany/lte 1.4, framework 1.7 |
| Framework API: `Modelo`, `Consulta`, `Rota`… | `composer require luany/pt` | luany/pt 1.0 |
| Validation rules: `obrigatorio\|texto` | always on | framework 1.8 |
| CLI messages | `LUANY_DEV_LANGUAGE=pt` | luany/cli 1.11 |

## Views

```dotenv
LUANY_SYNTAX=pt
```

```lte
@estende('layouts.main')

@secao('conteudo')
    @paracada($produtos as $produto)
        @se($produto->ativo)
            <li>{{ $produto->nome }}</li>
        @fimse
    @fimparacada
@fimsecao
```

The full table is in [Portuguese Directives](lte-portuguese.md).

## Models, queries and migrations

```bash
composer require luany/pt
```

```php
use Luany\Pt\Consulta;
use Luany\Pt\Modelo;

class Produto extends Modelo
{
    protected string $tabela       = 'produtos';
    protected array  $preenchiveis = ['nome', 'preco'];

    public function escopoAtivos(Consulta $c): void
    {
        $c->onde('ativo', '=', 1);
    }
}

$lista = Produto::ativos()->onde('preco', '<', 10)->ordenarPor('nome')->paginar(20);
```

```php
use Luany\Pt\Esquema;
use Luany\Pt\Tabela;

Esquema::para($pdo)->criar('produtos', function (Tabela $t) {
    $t->id();
    $t->chaveEstrangeira('categoria_id')->restrita()->apagarEmCascata();
    $t->texto('nome');
    $t->decimal('preco')->padrao(0);
    $t->marcasTempo();
});
```

`Modelo`, `Consulta` and `Tabela` are subclasses of `Model`, `ModelQuery` and `Blueprint`: the English methods are still there, and a query stays a `Consulta` through the whole chain. The complete list of names is in the [luany/pt README](https://github.com/luany-ecosystem/luany-pt).

## Routes and helpers

```php
use Luany\Pt\Rota;

Rota::get('/', [InicioControlador::class, 'index'])->nome('inicio');
Rota::recurso('produtos', ProdutoControlador::class);
```

Helpers: `vista()`, `redirecionar()`, `validar()`, `rota()`, `sessao()`, `valorAnterior()`, `abortar()`, `traduzir()`, `registo()`.

## Validation rules

Accepted in every project: `obrigatorio`, `texto`, `inteiro`, `entre`, `data`, `depois`… See [Validation](validation.md#portuguese-rule-names).

## CLI

```dotenv
LUANY_DEV_LANGUAGE=pt
```

```text
✓  Vista criada: views/pages/sobre.lte
✗  Utilização: luany make:controller <ControllerName>
```

Message codes (`LUANY-MAKE-003`) are the same in every language. `--json` and `--dry-run` work the same way — see [CLI](cli.md).
