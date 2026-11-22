# Comandos

A ferramenta de linha de comandos `luany` (pacote `luany/cli`) cria projetos, executa o servidor de desenvolvimento, gera código e gere a base de dados. Esta página lista todos os comandos, com os respetivos argumentos e opções.

## Instalação

Instale a CLI uma única vez, globalmente, com o Composer:

```bash
composer global require luany/cli
```

Certifique-se de que a pasta `bin` global do Composer está no seu `PATH`. A localização depende do sistema; pergunte-a ao Composer:

```bash
composer global config bin-dir --absolute
# por exemplo, adicione ao ~/.bashrc ou ~/.zshrc:
export PATH="$PATH:$HOME/.config/composer/vendor/bin"
```

Execute `luany` (ou `luany list`) sem argumentos para ver todos os comandos:

```bash
luany
```

### Deteção do projeto

A maioria dos comandos tem de ser executada dentro de um projeto Luany. A CLI sobe a partir da pasta atual até encontrar um `composer.json` que exija `luany/framework` ou `luany/core`, pelo que pode executar os comandos a partir de qualquer subpasta.

| Situação | Resultado |
|---|---|
| Fora de um projeto Luany | `This command must be run inside a Luany project.` (exit 1) |
| Projeto encontrado, mas falta `vendor/luany/framework` | Pede-lhe que execute `composer install` (exit 1) |

`new`, `doctor`, `about` e `list` funcionam em qualquer lado.

## Visão geral dos comandos

| Comando | Descrição |
|---|---|
| `new` | Cria um novo projeto Luany |
| `serve` | Inicia o servidor de desenvolvimento embutido do PHP |
| `dev` | Inicia o Luany Dev Engine (servidor PHP + live reload) |
| `make:controller` | Gera um controller |
| `make:model` | Gera um modelo |
| `make:migration` | Gera uma migration com timestamp |
| `make:seeder` | Gera um seeder |
| `make:middleware` | Gera um middleware |
| `make:provider` | Gera um service provider |
| `make:view` | Cria uma vista LTE (página, componente ou layout) |
| `make:request` | Gera uma classe de validação de form request |
| `make:test` | Gera um teste PHPUnit |
| `make:feature` | Gera uma funcionalidade CRUD completa |
| `migrate` | Executa todas as migrations pendentes |
| `migrate:rollback` | Reverte o último lote de migrations (`--step=N`: as últimas N migrations) |
| `migrate:status` | Mostra o estado de todas as migrations |
| `migrate:fresh` | Apaga todas as tabelas e volta a executar todas as migrations |
| `db:seed` | Executa os seeders da base de dados |
| `route:list` | Lista todas as rotas registadas |
| `route:cache` | Guarda a tabela de rotas em cache para produção |
| `route:clear` | Apaga a cache de rotas |
| `view:check` | Valida todas as vistas LTE sem as renderizar |
| `key:generate` | Gera e define a `APP_KEY` no `.env` |
| `cache:clear` | Limpa a cache das vistas compiladas |
| `env:encrypt` | Cifra o `.env` em `.env.{env}.encrypted` |
| `env:decrypt` | Decifra um ficheiro de ambiente cifrado |
| `env:edit` | Edita um ficheiro de ambiente cifrado no seu editor |
| `env:set` | Define valores num ficheiro de ambiente cifrado |
| `doctor` | Verifica o ambiente e o estado do projeto |
| `about` | Mostra as versões do projeto e dos pacotes |
| `list` | Lista todos os comandos disponíveis |

## Criar projetos

```bash
luany new blog
```

`luany new` executa `composer create-project luany/luany <name>` e depois configura o projeto. Num terminal interativo, faz todas as perguntas **antes** de o Composer arrancar, para que possa ir tratar de outra coisa enquanto as dependências são instaladas:

1. **Base de dados** — um menu numerado: `sqlite` (predefinido, nada a instalar), `mysql` ou `pgsql`. Um driver cuja extensão PDO do PHP esteja em falta aparece marcado com `(PHP extension missing)`.
2. **Credenciais** (apenas MySQL / PostgreSQL) — host, porta, nome da base de dados, utilizador e palavra-passe, com valores predefinidos.
3. **Git** — `Initialize a git repository? [Y/n]`, apenas quando o `git` está instalado.

Quando o Composer termina, a CLI escreve as definições `DB_*` no `.env`. Com SQLite, cria também `database/database.sqlite` e executa as migrations, deixando o projeto pronto para `luany dev`.

| Opção | Descrição |
|---|---|
| `--database=sqlite\|mysql\|pgsql` | Escolhe a base de dados sem perguntar |
| `--git` / `--no-git` | Inicializa (ou não) um repositório git sem perguntar |
| `--no-migrate` | Não executa as migrations depois de criar um projeto SQLite |
| `-n`, `--no-interaction` | Nunca pergunta: SQLite, sem git, credenciais predefinidas |

```bash
# Projeto PostgreSQL não interativo, com git
luany new shop --database=pgsql --git -n
```

Sem perguntas, MySQL e PostgreSQL recebem credenciais predefinidas: host `127.0.0.1`, porta `3306` / `5432`, utilizador `root` / `postgres`, palavra-passe vazia e uma base de dados com o nome do projeto (em minúsculas, com os caracteres não alfanuméricos substituídos por `_`). Crie essa base de dados, confirme o `.env` e depois execute `luany migrate`.

Os nomes de projeto podem conter letras, dígitos, `.`, `_` e `-`, e têm de começar por uma letra ou um dígito.

### Drivers PDO em falta

Se a extensão do PHP para o driver escolhido estiver em falta, `luany new` (bem como `migrate*`, `db:seed` e `doctor`) explica como a instalar, em vez de falhar com o "could not find driver" do PDO:

```bash
PHP's pdo_sqlite extension is not installed (needed for DB_CONNECTION=sqlite).
     Ubuntu / Debian:  sudo apt install php8.3-sqlite3
     Fedora / RHEL:    sudo dnf install php-pdo
     Windows:          enable extension=pdo_sqlite in php.ini
     macOS (Homebrew): included in the php formula
```

Nesse caso, a migração automática é ignorada e `luany migrate` fica nos próximos passos.

## Servidores de desenvolvimento

| Comando | Descrição |
|---|---|
| `luany serve [host] [port]` | Servidor embutido do PHP sobre `public/` (predefinido `localhost 8000`) |
| `luany dev [host] [port] [ws-port]` | Servidor PHP com observador de ficheiros e live reload no browser (predefinido `localhost 8000 35729`) |

Consulte [Servidor de desenvolvimento](dev-server.md) para mais detalhes.

## Geradores

Todos os geradores validam os nomes antes de escreverem o que quer que seja. Os nomes de classe aceitam StudlyCase, `snake_case` ou `kebab-case` (`blog_post` e `blog-post` dão ambos `BlogPost`). Caminhos como `../x`, nomes com espaços ou nomes que comecem por um dígito são rejeitados.

| Comando | Cria | Sufixo acrescentado | Subpastas |
|---|---|---|---|
| `make:controller Post` | `app/Controllers/PostController.php` | `Controller` | Sim: `Admin/Post` |
| `make:model Post` | `app/Models/Post.php` (tabela `posts`) | — | Não |
| `make:middleware Auth` | `app/Http/Middleware/AuthMiddleware.php` | `Middleware` | Sim |
| `make:provider Mail` | `app/Providers/MailServiceProvider.php` | `ServiceProvider` | Não |
| `make:request StoreUser` | `app/Http/Requests/StoreUserRequest.php` | `Request` | Sim |
| `make:test UserController` | `tests/UserControllerTest.php` | `Test` | Sim |
| `make:seeder User` | `database/seeders/UserSeeder.php` | `Seeder` | Não |

Se o ficheiro de destino já existir, o gerador não lhe toca e mostra um aviso (`make:seeder`, em vez disso, termina com erro).

Os controllers gerados em subpastas (`Admin/Post` → namespace `App\Controllers\Admin`) importam o `App\Controllers\Controller` base.

### make:migration

```bash
luany make:migration create_posts_table
luany make:migration add_slug_to_posts_table
```

O nome tem de estar em `snake_case` e começar por uma letra. O ficheiro é escrito em `database/migrations/{Y_m_d_His}_{name}.php` como uma migration de classe anónima que usa o schema builder. O template depende do nome:

| Padrão do nome | `up()` / `down()` gerados |
|---|---|
| `create_{table}_table` | `create('{table}', …)` com `id()` e `timestamps()` / `dropIfExists('{table}')` |
| `…_to_{table}_table`, `…_from_{table}_table`, `…_in_…`, `…_on_…` | `table('{table}', …)` em ambos os métodos |
| qualquer outro | `up()` e `down()` vazios |

Duas migrations com o mesmo nome são recusadas: declarariam a mesma classe. Consulte [Migrations e schema](migrations.md).

### make:view

```bash
luany make:view pages.about
luany make:view components.card component
luany make:view layouts.admin layout
```

O primeiro argumento é um nome de vista com pontos (segmentos de letras, dígitos, `_` ou `-`); o ficheiro é criado dentro de `views/`. O segundo argumento, opcional, escolhe o template: `page` (predefinido, estende `layouts.main`), `component` ou `layout`.

### make:feature

`luany make:feature Product name:string price:decimal` gera de uma só vez um modelo, uma migration, um controller, quatro vistas e um ficheiro de rotas. Tem página própria: [make:feature](make-feature.md).

## Comandos de base de dados

| Comando | Descrição |
|---|---|
| `luany migrate` | Executa todas as migrations pendentes em `database/migrations` |
| `luany migrate:rollback` | Reverte o último lote |
| `luany migrate:rollback --step=N` | Reverte as últimas N migrations (da mais recente para a mais antiga, independentemente do lote) |
| `luany migrate:status` | Tabela de migrations: executada ou não, e número do lote |
| `luany migrate:fresh` | Apaga **todas** as tabelas e depois executa todas as migrations |
| `luany db:seed` | Executa o `DatabaseSeeder` de `database/seeders` |

| Opção | Comando | Descrição |
|---|---|---|
| `--force` | `migrate:fresh` | Salta o pedido de confirmação (obrigatório em scripts e CI) |
| `--seed` | `migrate:fresh` | Executa o `DatabaseSeeder` depois de migrar |
| `--step=N` / `--step N` | `migrate:rollback` | Reverte as últimas N migrations em vez do último lote (exige luany/database 1.8 ou mais recente) |
| `--class=Name` / `--class Name` | `db:seed` | Executa um seeder específico em vez do `DatabaseSeeder` |

`migrate:fresh` pergunta `This drops ALL tables. Continue? [y/N]` e, por omissão, cancela — também quando a entrada não é um terminal, por isso passe sempre `--force` em automatismos. Sem `--step`, o `migrate:rollback` reverte um lote inteiro; com uma versão mais antiga do luany/database, `--step` termina com a indicação para executar `composer update luany/database`.

```bash
luany migrate:fresh --seed --force
luany migrate:rollback --step=2
luany db:seed --class=UserSeeder
```

### Definições de ligação

Os comandos de base de dados leem as mesmas definições que a aplicação, **sem a arrancar**:

| Variável | Utilização | Predefinição da CLI |
|---|---|---|
| `DB_CONNECTION` | `sqlite`, `mysql` ou `pgsql` | `sqlite` |
| `DB_NAME` | Nome da base de dados, ou caminho do ficheiro SQLite (relativo à raiz do projeto) | `database/database.sqlite` para SQLite, caso contrário `luany` |
| `DB_HOST` / `DB_PORT` | Endereço do servidor | `127.0.0.1` / `3306` (`5432` para pgsql) |
| `DB_USER` / `DB_PASS` | Credenciais | `root` (`postgres` para pgsql) / vazio |

Os valores são resolvidos por esta ordem:

1. Variáveis de ambiente reais (`$_ENV`, `$_SERVER`, `getenv()`) — as definições de Docker, CI e PaaS prevalecem.
2. O ficheiro `.env` do projeto.
3. As predefinições acima.

É a mesma precedência que a aplicação usa (o seu carregador de `.env` nunca sobrepõe variáveis existentes), pelo que a CLI comunica sempre com a mesma base de dados que a aplicação.

> **Nota** As predefinições da CLI coincidem com as do `DatabaseServiceProvider` do skeleton (`sqlite`, ficheiro `database/database.sqlite`). Os projetos em MySQL ou PostgreSQL têm de definir `DB_CONNECTION` no `.env`, como a aplicação já exige. No SQLite, os comandos ativam `PRAGMA foreign_keys = ON`, tal como a aplicação, e o `db:seed` / `migrate*` entregam aos modelos a ligação da CLI, para que funcionem em seeders e migrations (ver [Seeders](seeding.md#usar-o-query-builder-e-os-modelos)).

Se o `.env` existir mas não puder ser lido, o comando para com uma indicação sobre permissões. Se não houver `.env` nem nenhuma variável `DB_*` no ambiente, pede-lhe que execute `luany key:generate`.

## Rotas e vistas

### route:list

```bash
luany route:list
```

Carrega `routes/http.php` e depois todos os outros ficheiros `routes/*.php` (por ordem alfabética) sem arrancar o framework, e mostra uma tabela com métodos, URIs, ações (`PostController@show` ou `Closure`) e nomes.

### route:cache / route:clear

```bash
luany route:cache
luany route:clear
```

O `route:cache` arranca a aplicação tal como um pedido, lê todos os ficheiros `routes/*.php` e grava a tabela de rotas em `storage/cache/routes.php` (o `routeCachePath()` do Kernel), que o Kernel HTTP passa a carregar em vez dos ficheiros de rotas. Reconstrói sempre a partir dos ficheiros de rotas e termina com o código 1, listando as rotas, quando uma ação é uma closure. O `route:clear` apaga o ficheiro. Consulte [Rotas](routing.md#usar-a-cache-no-kernel).

### view:check

```bash
luany view:check
```

Valida todas as vistas `.lte` — estrutura de blocos, argumentos obrigatórios das diretivas e sintaxe PHP — sem as renderizar. Termina com o código 1 quando alguma vista tem erros. Consulte [view:check](view-check.md).

## Manutenção

### key:generate

```bash
luany key:generate
```

Cria o `.env` a partir do `.env.example`, se necessário, e escreve uma nova `APP_KEY` aleatória (`base64:` + 32 bytes aleatórios). O comando substitui uma chave existente sem perguntar. Os projetos criados com `luany new` ou `composer create-project` já têm uma chave.

### cache:clear

```bash
luany cache:clear
```

Apaga os templates compilados em `storage/cache/views` e indica quantos ficheiros foram removidos.

### env:encrypt / env:decrypt / env:edit / env:set

```bash
luany env:encrypt [--env=production] [--key=base64:…] [--force]
luany env:decrypt [--env=production] [--key=base64:…] [--force]
luany env:edit    [--env=production] [--key=base64:…]
luany env:set CHAVE=VALOR [CHAVE=VALOR …] [--env=production] [--key=base64:…]
```

Gerem [ficheiros de ambiente cifrados](configuration.md#ficheiros-de-ambiente-cifrados) (requerem luany/framework 1.6, cuja cifra os comandos usam).

| Comando | O que faz |
|---|---|
| `env:encrypt` | Cifra o `.env.{env}` (ou o `.env`) em `.env.{env}.encrypted`. Sem chave, gera uma e mostra-a uma única vez, com a variável a definir no servidor. Nunca substitui sem `--force`. |
| `env:decrypt` | Volta a escrever o ficheiro em claro (permissões 600). Nunca substitui sem `--force`. |
| `env:edit` | Abre o conteúdo decifrado no `$VISUAL` / `$EDITOR` (Bloco de Notas no Windows, senão nano ou vi) a partir de um ficheiro temporário privado, volta a cifrar e apaga o ficheiro temporário. |
| `env:set` | Altera valores sem nunca escrever um ficheiro em claro. Valores com espaços, `#`, `$` ou aspas são citados para serem lidos exatamente como foram escritos. |

A chave vem de `--key`; senão de `LUANY_ENV_KEY_{ENV}`; senão de `LUANY_ENV_KEY`.

### doctor

```bash
luany doctor
```

Verifica o ambiente e — dentro de um projeto — o estado do projeto:

| Âmbito | Verificações |
|---|---|
| Global | PHP ≥ 8.2; extensões `pdo`, o driver PDO de `DB_CONNECTION` (`pdo_sqlite` por omissão), `mbstring`, `openssl`, `json`; Composer; versão de `luany/cli` |
| Projeto | `.env`, `vendor/`, `vendor/luany/framework`, `database/migrations`, `APP_KEY`, `bootstrap/app.php`, `public/index.php`, `config/app.php` |
| Armazenamento | `storage/cache/views` e `storage/logs` têm permissão de escrita |
| Base de dados | Driver PDO instalado, ligação funcional (`from environment` quando as `DB_*` vêm de variáveis reais), tabela `_migrations` existe |

### about

```bash
luany about
```

Mostra o nome da aplicação, o ambiente, a flag de debug e o URL (primeiro das variáveis de ambiente reais, depois do `.env`, tal como o `env()` do framework), além das versões instaladas do PHP, `luany/framework`, `luany/core`, `luany/lte` (a partir do `composer.lock`) e `luany/cli`.

## --json, --dry-run e códigos das mensagens

Duas flags globais (luany/cli 1.11+):

```bash
luany make:feature Post title:string --dry-run   # plano e diff de cada ficheiro, nada é escrito
luany route:list --json                          # resultado legível por máquinas
```

O `--json` imprime um envelope, formato `luany.cli/1`:

```json
{
  "schema": "luany.cli/1",
  "ok": true,
  "command": "make:model",
  "cli_version": "1.12.0",
  "dry_run": true,
  "data": {},
  "plan": [{ "op": "create", "path": "app/Models/Post.php", "applied": false, "diff": "--- /dev/null\n+++ b/app/Models/Post.php\n…" }],
  "messages": [{ "code": "LUANY-MAKE-004", "level": "info", "text": "Would create app/Models/Post.php (dry run, nothing written)" }]
}
```

Em caso de falha, `ok` é `false`, o código de saída é diferente de zero e `error` traz `code`, `message` e `hint`. O `schema` é a versão do formato: campos novos mantêm-na; remover ou mudar um campo passa a `luany.cli/2`.

| Comando | `--dry-run` | Dados do `--json` |
|---|---|---|
| `make:*` (incluindo `make:feature`) | plano + diff de cada ficheiro | `make:feature`: `model`, `table`, `fields`, `routes` |
| `key:generate` | planeia a alteração do `.env` (nunca mostra a chave) | — |
| `cache:clear`, `route:clear` | planeia as remoções | `removed` |
| `route:list` | — | `routes` |
| `migrate:status` | — | `migrations` |
| `view:check` | — | `views`, `diagnostics` (`file`, `line`, `column`, `message`) |
| `about` | — | `app`, `versions` |

Os outros comandos recusam as duas flags (`LUANY-CLI-002` / `LUANY-CLI-003`) em vez de correrem a sério.

Cada mensagem tem um código estável (`LUANY-<ÁREA>-<NNN>`) que não muda entre versões nem línguas. Com `LUANY_DEV_LANGUAGE=pt` (no ambiente ou no `.env`) os textos ficam em português — veja [Programar em português](portuguese.md).

## Códigos de saída

Os comandos terminam com `0` em caso de sucesso e com um código diferente de zero em caso de erro: `1` para um comando desconhecido, um nome ou opção inválidos, um comando executado fora de um projeto ou vistas com erros; uma exceção inesperada (por exemplo, uma migration que falha) termina com o código não nulo do próprio PHP. Use-os para fazer falhar jobs de CI.
