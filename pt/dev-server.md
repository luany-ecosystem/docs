# Servidor de desenvolvimento

O Luany inclui dois servidores de desenvolvimento: `luany serve`, o servidor embutido do PHP sem mais nada, e `luany dev`, o Luany Dev Engine (LDE), que acrescenta um observador de ficheiros e live reload no browser. Esta página aborda ambos, os respetivos requisitos e o funcionamento do live reload.

## luany serve

```bash
luany serve                  # http://localhost:8000
luany serve 127.0.0.1 8080   # host e porta personalizados
```

O `serve` executa o servidor embutido do PHP com `public/` como raiz dos documentos. Os argumentos são posicionais:

| Argumento | Predefinição | Aceita |
|---|---|---|
| `host` | `localhost` | Nome de host, endereço IPv4 ou IPv6 entre parênteses retos (`[::1]`) |
| `port` | `8000` | Inteiro entre 1 e 65535 |

Pare-o com `Ctrl+C`. Use o `serve` quando não precisar de live reload ou não tiver o Node.js instalado.

## luany dev

```bash
luany dev
```

```bash
  ┌─────────────────────────────────────┐
  │   🚀  Luany Dev Engine  (LDE v1)    │
  └─────────────────────────────────────┘

  ✓  PHP server    →  http://localhost:8000
  ✓  Live reload   →  ws://localhost:35729
  ✓  CSS inject    →  active

  Press Ctrl+C to stop.
```

O `luany dev` arranca dois processos filhos e supervisiona-os:

1. **Servidor embutido do PHP** em `host:port`, a servir `public/`.
2. **Observador em Node.js** (`watcher.js`, incluído na CLI), que vigia os ficheiros do projeto e executa um servidor WebSocket.

O browser comunica diretamente com o PHP — não há proxy — e recebe as mensagens de recarregamento pelo WebSocket.

| Argumento | Predefinição | Descrição |
|---|---|---|
| `host` | `localhost` | Host do servidor PHP |
| `port` | `8000` | Porta do servidor PHP |
| `ws-port` | `35729` | Porta WebSocket do live reload |

```bash
luany dev localhost 8080          # porta HTTP personalizada
luany dev localhost 8000 35730    # porta WebSocket personalizada
```

As portas HTTP e WebSocket têm de ser diferentes. Antes de arrancar, o `dev` verifica se ambas estão livres; se uma estiver ocupada, para com uma sugestão como `luany dev localhost 8080`.

### Requisitos

| Requisito | Porquê |
|---|---|
| Node.js no `PATH` (`node` ou `nodejs`) | Executa o observador |
| `npm install` no projeto | Instala `chokidar` e `ws`, declarados no `package.json` do skeleton |
| `APP_ENV=development` (ou `local`) | Ativa o `LiveReloadMiddleware` |
| `LiveReloadMiddleware` no Kernel HTTP | Injeta o cliente no browser (registado no skeleton) |

```bash
npm install
luany dev
```

Se o Node.js ou os pacotes estiverem em falta, o `dev` para com uma mensagem clara (`Required Node.js packages not installed: chokidar, ws — Run: npm install`). O `package.json` do skeleton também associa `npm run dev` a `luany dev`.

Se o `.env` não contiver `APP_ENV=development`, o `dev` mostra um aviso, mas arranca na mesma.

> **Nota** O aviso só reconhece `development`; o middleware aceita também `APP_ENV=local`, pelo que nesse caso pode ignorar o aviso.

### O que é vigiado

| Caminho | Alteração | Ação no browser |
|---|---|---|
| `views/**/*.lte` | alteração, criação | Recarregamento completo |
| `app/**/*.php`, `routes/**/*.php`, `config/**/*.php` | alteração, criação | Recarregamento completo |
| `public/assets/**/*.css` | alteração, criação | **Injeção de CSS** (sem recarregar) |
| `public/assets/**/*` (outros ficheiros) | alteração, criação | Recarregamento completo |
| Qualquer ficheiro vigiado | eliminação | Recarregamento completo |

Os ficheiros e pastas ocultos, `node_modules`, a pasta `vendor/` de topo e `storage/cache` / `storage/logs` são ignorados. Os eventos passam por um debounce (40 ms): uma rajada de gravações produz uma única ação, um recarregamento completo prevalece sobre a injeção de CSS e, se várias folhas de estilo mudarem, todas são injetadas.

A **injeção de CSS** atualiza cada `<link rel="stylesheet">` cujo nome de ficheiro corresponda ao ficheiro alterado, acrescentando um cache-buster `?lde=<timestamp>`. Se nenhum link corresponder, a página é recarregada.

O terminal mostra cada evento:

```bash
[LDE] Changed: public/assets/css/app.css
[LDE] inject-css → app.css (1 client)
[LDE] Changed: views/pages/home.lte
[LDE] reload (1 client)
```

## LiveReloadMiddleware

O `Luany\Framework\Http\Middleware\LiveReloadMiddleware` liga o browser ao observador. O skeleton regista-o em primeiro lugar em `app/Http/Kernel.php`, para que veja a resposta final:

```php
protected array $middleware = [
    LiveReloadMiddleware::class, // live reload do `luany dev` — apenas em desenvolvimento + localhost
    CsrfMiddleware::class,
];
```

O middleware não faz nada a menos que se verifiquem **ambas** as condições:

- `APP_ENV` é `development` ou `local`;
- o pedido vem da interface de loopback (`REMOTE_ADDR` é `::1` ou começa por `127.`).

Quando está ativo:

1. Serve o cliente do browser em `/__luany_dev/client.js` (a partir de `LDE_CLIENT_PATH`, definido pelo `luany dev`, ou de `vendor/luany/cli` como alternativa).
2. Injeta `<script>window.__LDE_WS_PORT__ = 35729;</script>` e o `<script>` do cliente antes do último `</body>` da resposta.

Só são alteradas as respostas bem-sucedidas (2xx), e apenas quando o `Content-Type` é HTML ou não está definido (as vistas renderizadas com `view()` não o têm). JSON, ficheiros, redirecionamentos e páginas sem `</body>` ficam intactos.

O cliente volta a ligar-se automaticamente com back-off exponencial (até 5 segundos), pelo que uma página aberta continua a receber as alterações depois de reiniciar o `luany dev`.

> **Dica** Mantenha o `LiveReloadMiddleware` no Kernel em produção: com `APP_ENV=production` retorna de imediato. Consulte [Deploy](deployment.md).

## Segurança

- O servidor WebSocket escuta apenas em `127.0.0.1`.
- Só aceita ligações de páginas servidas por `localhost`, `127.0.0.1`, `::1`, `*.localhost` ou pelo host indicado ao `luany dev`; as outras origens são rejeitadas e registadas no log.
- O middleware nunca injeta nada em pedidos que não venham do loopback nem fora do ambiente de desenvolvimento.

Como tanto o WebSocket como o cliente usam o endereço de loopback, o live reload funciona num browser na mesma máquina. Outros dispositivos da sua rede podem abrir o site na mesma (com `luany dev 0.0.0.0`), mas sem live reload.

## Paragem e falhas

Prima `Ctrl+C` para parar ambos os processos. Se algum dos processos filhos terminar inesperadamente — por exemplo, se o servidor PHP não conseguir ocupar a porta —, o LDE para o outro e termina com o código de saída desse processo (ou `1`), para que os scripts possam detetar a falha.

## Notas para Windows

- O Node.js é localizado com `where`; certifique-se de que o `node` está no `PATH` do terminal que utiliza.
- `Ctrl+C` e `Ctrl+Break` são tratados nativamente; os processos filhos são parados com `taskkill /F /T`.
- São suportados caminhos com espaços (os processos são arrancados sem shell).
- Se o `variables_order` do seu PHP omitir `E` (comum em instalações WAMP), o observador herda na mesma o ambiente completo.
