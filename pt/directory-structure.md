# Estrutura de diretórios

Um projeto Luany novo é propositadamente pequeno: não contém código de demonstração que tenha de apagar primeiro. Esta página explica para que serve cada pasta.

## Visão geral

```
my-app/
├── app/
│   ├── Controllers/            Controllers HTTP (Controller.php é a classe base)
│   ├── Exceptions/Handler.php  Páginas de erro e reporte de exceções
│   ├── Http/
│   │   ├── Kernel.php          Middleware global
│   │   └── Middleware/         Middleware da aplicação (CsrfMiddleware)
│   ├── Models/                 Modelos da base de dados (User)
│   └── Providers/              Service providers (App, Database)
├── bootstrap/app.php           Cria e configura a aplicação
├── config/app.php              Nome, ambiente, URL, fuso horário, idioma
├── database/
│   ├── migrations/             Migrations do schema
│   ├── seeders/                Seeders da base de dados
│   └── database.sqlite         Base de dados SQLite (criada pelo instalador, ignorada pelo git)
├── public/                     Raiz web: index.php, .htaccess, assets/, favicon.svg
├── routes/http.php             Definição das rotas
├── storage/
│   ├── cache/views/            Templates LTE compilados
│   └── logs/                   Ficheiros de log
├── tests/                      Testes PHPUnit
├── views/                      Templates LTE
│   ├── components/flash.lte
│   ├── errors/404.lte, 500.lte
│   ├── layouts/main.lte
│   └── pages/home.lte
├── .env / .env.example
├── composer.json
└── package.json                Dependências do live reload para o `luany dev`
```

## A pasta `app`

O código da aplicação fica aqui, carregado automaticamente sob o namespace `App\` (PSR-4).

| Caminho | Conteúdo |
| --- | --- |
| `app/Controllers` | `App\Controllers\…` — consulte [Controllers](controllers.md). O `luany make:controller` escreve aqui |
| `app/Http/Kernel.php` | Estende o Kernel do framework; lista o [middleware global](middleware.md#middleware-global) |
| `app/Http/Middleware` | O seu middleware; o `luany make:middleware` escreve aqui |
| `app/Exceptions/Handler.php` | Gera as páginas de erro e reporta as exceções — consulte [Tratamento de erros](errors.md) |
| `app/Models` | [Modelos](models.md) |
| `app/Providers` | [Service providers](providers.md). O `DatabaseServiceProvider` abre a ligação a partir do `.env` |

Os geradores criam outras pastas à medida que são necessárias, como `app/Http/Requests` (`luany make:request`).

## `bootstrap/app.php`

Cria o container `Application`, carrega o `.env`, regista os service providers e associa o Kernel e o Handler de erros da aplicação. Tanto o `public/index.php` como os testes partem deste ficheiro. Consulte [Ciclo de vida do pedido](request-lifecycle.md).

## `config`

Ficheiros PHP que devolvem arrays, legíveis com `config('ficheiro.chave')`. Consulte [Configuração](configuration.md).

## `database`

Migrations, seeders e — no caso do SQLite — o ficheiro da base de dados. Consulte [Migrations](migrations.md) e [Seeders](seeding.md).

## `public`

A única pasta que o servidor web deve expor. O `index.php` é o front controller de todos os pedidos; os ficheiros estáticos ficam em `public/assets` e são referenciados com `asset('css/app.css')`.

## `routes`

O Kernel carrega primeiro o `routes/http.php` e depois **todos os outros ficheiros `*.php`** em `routes/`, por ordem alfabética. Dividir as rotas por área (`routes/admin.php`, `routes/api.php`) não exige qualquer registo — o `luany make:feature` tira partido disso. Consulte [Rotas](routing.md).

## `storage`

Ficheiros escritos pelo framework em tempo de execução. O servidor web tem de ter permissão de escrita aqui.

| Caminho | Escrito por |
| --- | --- |
| `storage/cache/views` | O motor LTE (templates compilados); limpo com `luany cache:clear` |
| `storage/logs` | O [logger](logging.md) — `luany-YYYY-MM-DD.log` |
| `storage/cache/routes.php` | O seu script de deploy, se usar a [cache de rotas](routing.md#usar-a-cache-no-kernel) — o Kernel lê-o quando existe |
| `storage/sessions` | Ficheiros de [sessão](session.md) — criada na primeira utilização com permissões `0700` |

As vistas compiladas, os logs e os ficheiros de sessão são ignorados pelo git no skeleton (`storage/sessions/` desde o skeleton 1.4.2).

## `views`

Templates LTE referenciados com notação de pontos: `view('pages.home')` gera `views/pages/home.lte`. O handler de erros do skeleton usa `views/errors/404.lte` e `views/errors/500.lte`. Consulte [Sintaxe dos templates](lte-syntax.md).

## `lang`

Não é criada por predefinição. Adicione ficheiros `lang/{locale}.php` quando precisar de [traduções](localization.md).

## `tests`

Testes PHPUnit sob o namespace `Tests\`. O skeleton inclui um teste funcional que arranca o Kernel real e pede a página inicial:

```bash
composer test
```
