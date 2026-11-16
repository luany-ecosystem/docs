# Instalação

Esta página aborda os requisitos, a criação de um novo projeto com a CLI do Luany e a sua execução em ambiente local.

## Requisitos

| Requisito | Notas |
| --- | --- |
| PHP 8.2+ | Com `pdo_sqlite` para a configuração predefinida, ou `pdo_mysql` / `pdo_pgsql` |
| Composer | Instala os pacotes do framework |
| Node.js | Só é necessário para o live reload (`luany dev`) |

Execute `luany doctor` a qualquer momento para verificar o seu ambiente e o estado de um projeto.

## Instalar a CLI

O comando `luany` é instalado uma única vez, globalmente, com o Composer:

```bash
composer global require luany/cli
```

Certifique-se de que a pasta `bin` global do Composer está no seu `PATH`:

```bash
# macOS / Linux
export PATH="$PATH:$HOME/.composer/vendor/bin"

# Windows
%USERPROFILE%\AppData\Roaming\Composer\vendor\bin
```

## Criar um projeto

```bash
luany new my-app
```

O `luany new` faz as perguntas **antes** de o Composer arrancar, por isso pode afastar-se enquanto as dependências são instaladas:

```
  Database
   ›  1  sqlite     file-based, nothing to install (recommended to start)
      2  mysql      MySQL / MariaDB server
      3  pgsql      PostgreSQL server
  Choose 1-3 or a name [sqlite]:
  Initialize a git repository? [Y/n]:
```

- Os drivers cuja extensão PDO está em falta aparecem na lista com a indicação `(PHP extension missing)`.
- Para **MySQL** ou **PostgreSQL** são pedidos o host, a porta, o nome da base de dados, o utilizador e a palavra-passe. Os valores predefinidos são `127.0.0.1`, a porta `3306` / `5432`, uma base de dados com o nome do projeto (em minúsculas, com os restantes caracteres substituídos por `_`) e o utilizador `root` / `postgres`.
- A pergunta sobre o git só aparece quando o `git` está instalado.

Em seguida, o comando executa `composer create-project luany/luany my-app` e configura a base de dados:

| Base de dados | O que o `luany new` faz |
| --- | --- |
| SQLite | Escreve `DB_CONNECTION=sqlite`, cria `database/database.sqlite` e executa as migrations — a aplicação funciona de imediato |
| MySQL / PostgreSQL | Escreve os valores `DB_*` no `.env`. Crie a base de dados manualmente e depois execute `luany migrate` |

Se a extensão PDO do driver escolhido estiver em falta, o comando explica como instalá-la (apt, dnf, php.ini) e não executa as migrations.

### Opções

| Opção | Descrição |
| --- | --- |
| `--database=sqlite\|mysql\|pgsql` | Escolhe a base de dados sem perguntar |
| `--git` / `--no-git` | Inicializa (ou não) um repositório git |
| `--no-migrate` | Não executa as migrations para SQLite |
| `-n`, `--no-interaction` | Não pergunta nada: SQLite, migrations executadas, sem git (exceto com `--git`) |

```bash
luany new my-app --database=pgsql --git
luany new my-app -n
```

Os nomes de projeto podem conter letras, dígitos, `.`, `_` e `-`, e têm de começar por uma letra ou um dígito.

### Sem a CLI

Também pode criar um projeto diretamente com o Composer:

```bash
composer create-project luany/luany my-app
cd my-app
luany migrate
```

O script de pós-instalação do skeleton copia `.env.example` para `.env`, gera uma nova `APP_KEY` e cria o ficheiro da base de dados SQLite. Ao contrário do `luany new`, não executa as migrations.

### Versões

Os novos projetos partem do skeleton 1.4.2, com o `luany/framework` 1.4 e o `luany/core` 1.4. Para atualizar um projeto existente:

```bash
composer update luany/framework luany/core
```

Os projetos criados antes do skeleton 1.4.2 devem também acrescentar `/storage/sessions/` ao `.gitignore` e apagar a redefinição vazia de `$dontReport` em `app/Exceptions/Handler.php` — consulte [Tratamento de erros](errors.md#reporte-de-exceções).

## Executar a aplicação

```bash
cd my-app
luany dev
```

O `luany dev` inicia o servidor embutido do PHP em `http://localhost:8000`, juntamente com um observador de ficheiros que recarrega o browser quando altera vistas, ficheiros PHP, rotas ou configuração (o CSS é injetado sem recarregar a página inteira). Precisa do Node.js e das dependências do observador:

```bash
npm install
```

O live reload só fica ativo quando `APP_ENV=development` e o pedido vem da máquina local. Consulte [Servidor de desenvolvimento](dev-server.md).

Sem Node.js, use o servidor PHP simples:

```bash
luany serve                    # http://localhost:8000
luany serve localhost 8080     # host e porta personalizados
```

## Configuração do servidor web

Aponte a raiz de documentos (document root) do servidor web para a pasta `public/` — nunca para a raiz do projeto, que contém o `.env`.

O **Apache** funciona sem configuração adicional: o `public/.htaccess` encaminha para o `index.php` todos os pedidos que não correspondam a um ficheiro real, reencaminha o cabeçalho `Authorization`, bloqueia ficheiros ocultos (dotfiles) e define cabeçalhos de segurança básicos. Não tem um `RewriteBase` fixo, pelo que a aplicação também funciona numa subpasta.

O **Nginx** precisa da regra habitual de front controller:

```nginx
root /var/www/my-app/public;
index index.php;

location / {
    try_files $uri $uri/ /index.php?$query_string;
}

location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    fastcgi_pass unix:/run/php/php8.2-fpm.sock;
}
```

Garanta que o utilizador do servidor web tem permissão de escrita em `storage/`: as vistas são compiladas para `storage/cache/views`, os logs vão para `storage/logs` e as sessões para `storage/sessions`.

## Próximos passos

- [Configuração](configuration.md) — `.env` e `config/app.php`
- [Estrutura de diretórios](directory-structure.md) — onde fica cada coisa
- [Rotas](routing.md) — adicione as suas primeiras páginas
- [make:feature](make-feature.md) — gere uma funcionalidade CRUD completa com um único comando
