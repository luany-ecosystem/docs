# Deploy

Esta página descreve como colocar uma aplicação Luany em produção: definições de ambiente, dependências, caches, permissões de ficheiros, configuração do servidor web para Apache e Nginx, HTTPS atrás de proxies e migrations.

## Requisitos

- PHP 8.2 ou mais recente com `pdo`, o driver PDO da sua base de dados (`pdo_sqlite`, `pdo_mysql` ou `pdo_pgsql`), `mbstring`, `openssl` e `json`.
- Composer.
- Um servidor web (Apache ou Nginx) com PHP-FPM, ou qualquer servidor capaz de encaminhar todos os pedidos para `public/index.php`.

O Node.js **não** é necessário em produção — serve apenas o `luany dev`.

Execute `luany doctor` no servidor para verificar de uma só vez o PHP, as extensões, o `.env`, a `APP_KEY`, as permissões de armazenamento e a ligação à base de dados (consulte [Comandos](cli.md#doctor)).

## Ambiente

Crie o `.env` no servidor (é ignorado pelo git) a partir do `.env.example` e defina, pelo menos:

```env
APP_NAME="My App"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com
APP_KEY=base64:...
APP_TIMEZONE=UTC

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=myapp
DB_USER=myapp
DB_PASS=secret
```

| Definição | Porquê |
|---|---|
| `APP_ENV=production` | Desativa o live reload e os comportamentos exclusivos de desenvolvimento |
| `APP_DEBUG=false` | Esconde a página de erro de debug (stack traces, caminhos de ficheiros) e desativa `@dump` / `@dd` |
| `APP_KEY` | Gere-a com `luany key:generate` se o ficheiro for novo |
| `DB_CONNECTION` | Defina-a sempre explicitamente |

As variáveis de ambiente reais prevalecem sobre o `.env` — tanto na aplicação como na CLI —, pelo que as plataformas de containers podem injetar as definições em vez de distribuírem um ficheiro. `APP_ENV` e `APP_URL` são obrigatórias: o bootstrap do skeleton lança uma exceção se faltarem. Consulte [Configuração](configuration.md).

Proteja o ficheiro:

```bash
chmod 640 .env
```

## Instalar as dependências

Instale sem os pacotes de desenvolvimento e com um autoloader otimizado:

```bash
composer install --no-dev --optimize-autoloader --no-interaction
```

O skeleton já define `optimize-autoloader` no `composer.json`. Se fizer deploy a partir de um artefacto de build, execute o mesmo comando no passo de build e inclua a pasta `vendor/`.

## Migrations da base de dados

Execute as migrations pendentes em cada deploy:

```bash
luany migrate
```

A CLI lê as `DB_*` primeiro do ambiente real e depois do `.env`, pelo que usa a mesma base de dados que a aplicação. O `luany` tem de estar instalado no servidor (`composer global require luany/cli`) ou adicionado como dependência.

> **Aviso** Nunca execute `luany migrate:fresh` em produção — apaga todas as tabelas. Pede confirmação, a menos que seja indicado `--force`, e a resposta predefinida é "Não".

Verifique o que foi executado com `luany migrate:status`; reverta o último lote com `luany migrate:rollback`.

## Caches

### Vistas

O LTE compila os templates para PHP em `storage/cache/views`. Em produção (`APP_DEBUG=false`), um template só é recompilado quando o ficheiro de origem é mais recente do que o ficheiro compilado, pelo que os templates novos são detetados depois de um deploy. Ainda assim, limpe a cache para remover ficheiros compilados obsoletos:

```bash
luany cache:clear
```

Ative o OPcache em produção; os templates compilados são invalidados no OPcache quando são reescritos.

### Rotas

O Kernel HTTP carrega `routes/http.php` e todos os outros ficheiros `routes/*.php` em cada pedido. Não lê nenhuma cache de rotas, e a CLI não tem um comando `route:cache`. A API `Route::cache()` do `luany/core` destina-se a front controllers personalizados e a workers de longa duração (consulte [Rotas](routing.md#cache-de-rotas)). Com o OPcache ativo, carregar os ficheiros de rotas tem um custo baixo.

## Permissões de ficheiros

O utilizador do servidor web (por exemplo, `www-data`) tem de conseguir escrever em:

| Caminho | Utilização |
|---|---|
| `storage/cache/views` | Templates compilados |
| `storage/logs` | Ficheiros de log diários |
| `storage/sessions` | Ficheiros de sessão (criados na primeira utilização) |
| `storage/rate-limits` (ou o seu caminho) | `FileRateLimiter`, se for usado |
| `database/` | Só SQLite: o ficheiro da base de dados **e** a respetiva pasta (o SQLite escreve um journal ao lado) |

Tudo o resto pode ser só de leitura:

```bash
sudo chown -R deploy:www-data /var/www/myapp
sudo find /var/www/myapp -type d -exec chmod 755 {} \;
sudo find /var/www/myapp -type f -exec chmod 644 {} \;
sudo chmod -R ug+rwX /var/www/myapp/storage
sudo chmod 640 /var/www/myapp/.env
```

## Servidor web

Aponte a raiz dos documentos para `public/`. Só `public/` pode estar acessível a partir da web: a raiz do projeto contém o `.env`, `vendor/`, `storage/` e o seu código-fonte.

### Apache

O skeleton inclui `public/.htaccess`. Com o `mod_rewrite` ativo e `AllowOverride All` para a pasta, este ficheiro:

- encaminha para `index.php` todos os pedidos que não correspondam a um ficheiro ou pasta existente;
- passa o cabeçalho `Authorization` ao PHP;
- define `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` e `Permissions-Policy` (com o `mod_headers`);
- bloqueia o acesso a dotfiles e desativa a listagem de pastas.

```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/myapp/public

    <Directory /var/www/myapp/public>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Descomente o cabeçalho `Strict-Transport-Security` no `.htaccess` assim que o site funcionar apenas em HTTPS, e o exemplo de `Content-Security-Policy` depois de o adaptar aos seus assets.

### Nginx

O Nginx não lê o `.htaccess`; traduza as suas regras para o bloco `server`:

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/myapp/public;
    index index.php;

    charset utf-8;
    client_max_body_size 10m;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param HTTP_AUTHORIZATION $http_authorization;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    # Nunca servir dotfiles (.env, .git, ...)
    location ~ /\.(?!well-known) {
        deny all;
    }
}
```

Ajuste o socket do PHP-FPM à sua versão do PHP.

## HTTPS e proxies

Sirva o site apenas por HTTPS e redirecione HTTP para HTTPS no servidor web ou no load balancer.

Se o TLS terminar num load balancer ou reverse proxy, o PHP vê HTTP simples vindo do endereço do proxy. Há duas definições que resolvem isso:

1. Confiar no proxy, para que `$request->ip()` e `$request->isSecure()` usem `X-Forwarded-For` / `X-Forwarded-Proto` — por exemplo, em `AppServiceProvider::boot()`:

   ```php
   use Luany\Core\Http\Request;

   Request::setTrustedProxies(['10.0.0.0/8']);
   ```

2. Forçar cookies de sessão seguros — a deteção automática só olha para a flag HTTPS do próprio servidor:

   ```php
   // config/session.php
   return [
       'secure' => true,
   ];
   ```

Defina `APP_URL` com o URL público `https://`, para que `url()` e `asset()` gerem links corretos.

## Checklist de deploy

- [ ] `.env` presente com `APP_ENV=production`, `APP_DEBUG=false`, `APP_URL`, `APP_KEY` e definições `DB_*` explícitas; `chmod 640`.
- [ ] `composer install --no-dev --optimize-autoloader`.
- [ ] A raiz dos documentos é `public/`; `.htaccess` do Apache ativo ou regras equivalentes no Nginx.
- [ ] `storage/` (e `database/` no caso do SQLite) com permissão de escrita para o servidor web.
- [ ] `luany migrate` executado.
- [ ] `luany cache:clear` executado; OPcache ativo.
- [ ] HTTPS obrigatório; proxies de confiança e cookie de sessão `secure` configurados quando atrás de um proxy.
- [ ] `luany doctor` passa sem erros.
- [ ] Logs em `storage/logs` monitorizados e com rotação.

Consulte a checklist de [Segurança](security.md) para os pontos ao nível da aplicação.
