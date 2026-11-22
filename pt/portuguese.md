# Programar em português

O Luany pode ser escrito em português de ponta a ponta — vistas, API do framework, regras de validação e mensagens da CLI — com PHP normal por baixo. Cada camada liga-se à parte, o inglês continua a funcionar e os dois podem misturar-se.

| Camada | Como ativar | Desde |
|---|---|---|
| Vistas: `@se`, `@paracada`, `@secao`… | `LUANY_SYNTAX=pt` | luany/lte 1.4, framework 1.7 |
| API do framework: `Modelo`, `Consulta`, `Rota`… | `composer require luany/pt` | luany/pt 1.0 |
| Regras de validação: `obrigatorio\|texto` | sempre ativas | framework 1.8 |
| Mensagens da CLI | `LUANY_DEV_LANGUAGE=pt` | luany/cli 1.11 |

## Vistas

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

A tabela completa está em [Diretivas em português](lte-portuguese.md).

## Modelos, consultas e migrations

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

`Modelo`, `Consulta` e `Tabela` são subclasses de `Model`, `ModelQuery` e `Blueprint`: os métodos ingleses continuam lá, e uma consulta continua a ser uma `Consulta` em toda a cadeia. A lista completa de nomes está no [README do luany/pt](https://github.com/luany-ecosystem/luany-pt).

## Rotas e helpers

```php
use Luany\Pt\Rota;

Rota::get('/', [InicioControlador::class, 'index'])->nome('inicio');
Rota::recurso('produtos', ProdutoControlador::class);
```

Helpers: `vista()`, `redirecionar()`, `validar()`, `rota()`, `sessao()`, `valorAnterior()`, `abortar()`, `traduzir()`, `registo()`.

## Regras de validação

Aceites em qualquer projeto: `obrigatorio`, `texto`, `inteiro`, `entre`, `data`, `depois`… Veja [Validação](validation.md#regras-em-português).

## CLI

```dotenv
LUANY_DEV_LANGUAGE=pt
```

```text
✓  Vista criada: views/pages/sobre.lte
✗  Utilização: luany make:controller <ControllerName>
```

Os códigos das mensagens (`LUANY-MAKE-003`) são iguais em todas as línguas. O `--json` e o `--dry-run` funcionam da mesma forma — veja [CLI](cli.md).
