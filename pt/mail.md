# Email

O luany/framework 1.10+ envia emails escritos como vistas LTE.

```php
mailer()->to($user->email, $user->nome)
    ->subject('Bem-vindo')
    ->send('emails.bem-vindo', ['nome' => $user->nome, 'url' => url('/painel')]);
```

O `app('mailer')` é o mesmo objeto. Também: `cc()`, `bcc()`, `replyTo()`, `from()`, `attach($caminho, $nome)` e `sendText($texto, $html)` para um email sem vista.

## Vistas

`views/emails/bem-vindo.lte` é o corpo HTML. Se existir `views/emails/bem-vindo_text.lte`, é o corpo em texto simples; senão, o texto é tirado do HTML (os links ficam `texto (url)`). As variáveis são escapadas como em qualquer vista, `$assunto` (e `$subject`) tem o assunto, e as diretivas em português funcionam com `LUANY_SYNTAX=pt`.

```lte
<h1>Olá {{ $nome }}</h1>
<p><a href="{{ $url }}">Abrir o painel</a></p>
```

## Configuração

Por omissão os emails vão para o log (`storage/logs`) — nada a configurar em desenvolvimento. Para enviar a sério, no `.env`:

```dotenv
MAIL_MAILER=smtp
MAIL_FROM_ADDRESS=ola@exemplo.ao
MAIL_FROM_NAME="${APP_NAME}"
MAIL_HOST=smtp.exemplo.ao
MAIL_PORT=587
MAIL_USERNAME=...
MAIL_PASSWORD=...
MAIL_ENCRYPTION=tls
```

O `MAIL_ENCRYPTION` é `tls` (STARTTLS obrigatório — porta 587), `ssl` (porta 465) ou `none` (só para relays locais). As mesmas chaves estão em `config/mail.php`. Em produção, guarde a palavra-passe num [ficheiro de ambiente cifrado](configuration.md#ficheiros-de-ambiente-cifrados).

| `MAIL_MAILER` | Entrega |
|---|---|
| `log` (omissão) | escrito no log |
| `smtp` | enviado pelo servidor SMTP |
| `array` | guardado em memória — para testes: `mailer()->transport()->messages` |

## Erros

As falhas lançam `Luany\Framework\Mail\MailException` com um código estável:

| Código | Significado |
|---|---|
| `LUANY-MAIL-001` | endereço inválido |
| `LUANY-MAIL-002` | quebra de linha num cabeçalho (injeção de cabeçalhos) |
| `LUANY-MAIL-003` | o servidor SMTP recusou ou não respondeu |
| `LUANY-MAIL-004` | sem destinatário, assunto ou corpo |
| `LUANY-MAIL-005` | configuração (sem remetente, mailer desconhecido, host vazio) |
| `LUANY-MAIL-006` | anexo não encontrado |

As palavras-passe nunca aparecem nestas mensagens nem no log.
