# Email

luany/framework 1.10+ sends email written as LTE views.

```php
mailer()->to($user->email, $user->name)
    ->subject('Welcome')
    ->send('emails.welcome', ['name' => $user->name, 'url' => url('/dashboard')]);
```

`app('mailer')` is the same object. Also: `cc()`, `bcc()`, `replyTo()`, `from()`, `attach($path, $name)` and `sendText($text, $html)` for an email without a view.

## Views

`views/emails/welcome.lte` is the HTML body. If `views/emails/welcome_text.lte` exists it is the plain-text body; otherwise the text is taken from the HTML (links become `text (url)`). Variables are escaped as in any view, `$subject` holds the subject, and the Portuguese directives work when `LUANY_SYNTAX=pt`.

```lte
<h1>Hello {{ $name }}</h1>
<p><a href="{{ $url }}">Open your dashboard</a></p>
```

## Configuration

By default emails are written to the log (`storage/logs`) — nothing to configure in development. To send for real, in `.env`:

```dotenv
MAIL_MAILER=smtp
MAIL_FROM_ADDRESS=hello@example.com
MAIL_FROM_NAME="${APP_NAME}"
MAIL_HOST=smtp.example.com
MAIL_PORT=587
MAIL_USERNAME=...
MAIL_PASSWORD=...
MAIL_ENCRYPTION=tls
```

`MAIL_ENCRYPTION` is `tls` (STARTTLS, required — port 587), `ssl` (port 465) or `none` (local relays only). The same keys live in `config/mail.php`. Keep the password in an [encrypted environment file](configuration.md#encrypted-environment-files) in production.

| `MAIL_MAILER` | Delivery |
|---|---|
| `log` (default) | written to the log |
| `smtp` | sent through the SMTP server |
| `array` | kept in memory — for tests: `mailer()->transport()->messages` |

## Errors

Failures throw `Luany\Framework\Mail\MailException` with a stable code:

| Code | Meaning |
|---|---|
| `LUANY-MAIL-001` | invalid address |
| `LUANY-MAIL-002` | line break in a header value (header injection) |
| `LUANY-MAIL-003` | the SMTP server refused or could not be reached |
| `LUANY-MAIL-004` | no recipient, subject or body |
| `LUANY-MAIL-005` | configuration (no sender, unknown mailer, empty host) |
| `LUANY-MAIL-006` | attachment not found |

Passwords never appear in these messages or in the log.
