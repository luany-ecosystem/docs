# Validation

Luany ships with a dependency-free validator. The `validate()` helper covers the common case — validate, redirect back with errors on failure, return clean data on success — and the `Validator` class gives full control.

## Quick start

```php
public function store(Request $request): Response
{
    $data = validate($request->body(), [
        'name'     => 'required|string|min:2|max:255',
        'email'    => 'required|email|unique:users,email',
        'password' => 'required|string|min:8|confirmed',
        'role'     => 'required|in:admin,editor,viewer',
        'active'   => 'boolean',
    ]);

    User::create($data);   // only the validated fields

    return redirect('/users');
}
```

On success, `validate()` returns **only the fields that have rules**. On failure it throws a `ValidationException`, which the Kernel turns into a response:

| Request | Response |
| --- | --- |
| Browser form | Errors and old input are flashed to the session, then a redirect |
| JSON (`Accept: application/json` or AJAX) | `422` with `{"message": "The given data was invalid.", "errors": {...}}` |

The redirect goes to the third argument when given, otherwise to the `Referer` **if it points to the current host**, otherwise to `/`:

```php
validate($request->body(), $rules, '/users/create');
```

Old input never includes `password*` fields, `_token`, `csrf_token` or `_method`. Add your own keys at boot:

```php
use Luany\Framework\Exceptions\ValidationException;

ValidationException::$dontFlash[] = 'credit_card';
```

## Displaying errors

After the redirect, errors are in the session under `errors` (an array of messages per field) and old input is available through `old()`:

```lte
@php($errors = session('errors', []))

<form method="POST" action="/users">
    @csrf
    <input name="email" value="{{ old('email') }}">
    @if(!empty($errors['email']))
        <p class="error">{{ $errors['email'][0] }}</p>
    @endif
    <button>Save</button>
</form>
```

Both are [flash data](session.md#flash-data): they last for the next request only.

A JSON error response looks like this:

```json
{
    "message": "The given data was invalid.",
    "errors": {
        "email": ["The email field must be a valid email address."]
    }
}
```

## Using the Validator directly

```php
use Luany\Framework\Validation\Validator;

$v = Validator::make($request->body(), [
    'title' => 'required|string|max:255',
]);

if ($v->fails()) {
    return Response::json(['errors' => $v->errors()], 422);
}

$data = $v->validated();
```

| Method | Returns |
| --- | --- |
| `Validator::make($data, $rules)` | A validator that has already run |
| `fails()` / `passes()` | `bool` |
| `errors()` | `['field' => ['message', ...]]` |
| `validated()` | The validated fields only |

## Rule syntax

Rules are a pipe-separated string or an array. Use an array for regular expressions that contain `|` and for custom rule objects or closures:

```php
'code'  => 'required|string|max:10',
'phone' => ['required', 'regex:/^\+?[0-9 ]{7,15}$/'],
```

An unknown rule name throws an `InvalidArgumentException` — a typo never silently disables validation.

## Available rules

| Rule | Passes when |
| --- | --- |
| `required` | Present and not `null`, `''` or `[]` |
| `sometimes` | Flag: validate the field only when it is present in the input |
| `nullable` | Flag: `null` or `''` is accepted as `null`; other rules are skipped |
| `string` | A string |
| `email` | A valid email address |
| `integer` | An int, or an integer string such as `"42"` or `"-3"` (no surrounding spaces) |
| `numeric` | A number or numeric string |
| `boolean` | `true`, `false`, `1`, `0`, `"1"`, `"0"`, `"true"`, `"false"`, `"on"`, `"off"`, `"yes"`, `"no"` |
| `array` | An array |
| `date` | A date `strtotime()` understands |
| `after:{date\|field}` | A date after the given date or another field's value |
| `before:{date\|field}` | A date before the given date or field |
| `after_or_equal:{date\|field}` / `before_or_equal:{date\|field}` | Inclusive versions |
| `min:{n}` / `max:{n}` | Size at least / at most `n` (see [sizes](#sizes)) |
| `between:{min},{max}` | Size within the range, inclusive |
| `digits:{n}` | Exactly `n` digits |
| `in:{a},{b},...` | One of the listed values (compared as strings) |
| `not_in:{a},{b},...` | None of the listed values |
| `regex:{pattern}` / `not_regex:{pattern}` | Matches / does not match the pattern |
| `url` | An absolute `http` or `https` URL |
| `confirmed` | Identical to `{field}_confirmation` |
| `same:{field}` / `different:{field}` | Equal to / different from another field |
| `unique:{table},{column},{ignore},{ignoreColumn}` | No row has this value — see [database rules](#database-rules) |
| `exists:{table},{column}` | A row has this value; for arrays, every element exists |
| `file` | A successful upload — see [file uploads](#file-uploads) |
| `image` | An uploaded JPEG, PNG, GIF, BMP or WebP image (not SVG) |
| `mimes:{ext},{ext},...` | An uploaded file whose detected type matches one of the extensions |

### Empty values

Most rules **skip** `null` and `''` — combine them with `required` to make a field mandatory. `'email' => 'email'` accepts an empty field but rejects `"abc"`.

### `nullable`, `sometimes` and `boolean`

These rules also shape the validated data:

| Input | `'bio' => 'nullable\|string'` | `'bio' => 'sometimes\|string'` | `'active' => 'boolean'` |
| --- | --- | --- | --- |
| Field absent | not in `validated()` | not validated, not in `validated()` | `false` |
| `''` | `null` | validated (`''` passes `string`) | `false` |
| `"on"` / `"1"` | `"on"` / `"1"` | `"on"` / `"1"` | `true` |

A `boolean` field is always cast to `bool`, and an absent one becomes `false` — exactly what an unchecked checkbox needs. Add `sometimes` to leave an absent boolean out (for partial updates).

### Sizes

`min`, `max` and `between` measure:

| Value | Measured by |
| --- | --- |
| Uploaded file | Its size in **kilobytes** |
| Array | Its number of items |
| `int` or `float` | Its value |
| Numeric string, when the field also has `numeric` or `integer` | Its value |
| Any other string | Its length in characters |

Form input always arrives as strings, so add `numeric` or `integer` to compare a number by value:

```php
'age'  => 'required|integer|min:18',     // "17" fails: compared as 17
'name' => 'required|string|max:255',     // "300" passes: 3 characters
'pin'  => 'required|string|min:4',       // "1234" passes: 4 characters
```

### Dates

`after`, `before` and their `_or_equal` variants accept a date string or the name of another field:

```php
'starts_at' => 'required|date|after:today',
'ends_at'   => 'required|date|after:starts_at',
```

## Portuguese rule names

Every rule also has a Portuguese name, accepted in any project (no package needed). Both languages can be mixed:

```php
validate($request->body(), [
    'nome'  => 'obrigatorio|texto|max:120',
    'idade' => 'required|inteiro|entre:18,99',
]);
```

| English | Portuguese | English | Portuguese |
|---|---|---|---|
| `required` | `obrigatorio` | `in` / `not_in` | `em` / `nao_em` |
| `nullable` | `opcional` | `between` | `entre` |
| `sometimes` | `quando_presente` | `digits` | `digitos` |
| `string` | `texto` | `confirmed` | `confirmado` |
| `integer` / `numeric` | `inteiro` / `numerico` | `same` / `different` | `igual` / `diferente` |
| `boolean` | `booleano` | `unique` / `exists` | `unico` / `existe` |
| `array` | `lista` | `file` | `ficheiro` (or `arquivo`) |
| `date` | `data` | `image` | `imagem` |
| `after` / `before` | `depois` / `antes` | `mimes` | `tipos` |
| `after_or_equal` / `before_or_equal` | `depois_ou_igual` / `antes_ou_igual` | `not_regex` | `nao_regex` |

`email`, `url`, `min`, `max` and `regex` are the same in both languages. A rule you register with `Validator::extend()` under one of these names keeps its meaning. See [Programming in Portuguese](portuguese.md).

## File uploads

`file`, `image` and `mimes` validate a `$_FILES`-style entry (`name`, `type`, `tmp_name`, `error`, `size`). Uploads are not part of `body()`, so add them to the data you validate:

```php
public function update(Request $request): Response
{
    $data = validate($request->body() + $request->files(), [
        'name'   => 'required|string|max:255',
        'avatar' => 'nullable|image|max:2048',          // optional, up to 2 MB
        'cv'     => 'required|file|mimes:pdf|max:5120', // required, PDF, up to 5 MB
    ]);

    if ($data['avatar'] !== null) {
        $name = bin2hex(random_bytes(16));   // never reuse the client's file name
        move_uploaded_file($data['avatar']['tmp_name'], base_path('storage/uploads/' . $name));
    }
    // ...
}
```

- An empty file input (`UPLOAD_ERR_NO_FILE`) counts as no value: `required` fails, `nullable` gives `null`, and the other rules skip it. Any other upload error fails `file`, `image` and `mimes`.
- The type is detected from the file **contents** with `finfo`, never from the client's `Content-Type`. Without the `fileinfo` extension, the client file name's extension is used instead — enable `fileinfo` in production.
- `mimes` knows `jpg`, `jpeg`, `png`, `gif`, `bmp`, `webp`, `svg`, `pdf`, `txt`, `csv`, `json`, `xml`, `zip`, `mp3`, `mp4`, `doc`, `docx`, `xls` and `xlsx`. Any other extension is matched against the client file name only.
- `tmp_name` must pass `is_uploaded_file()`, so a crafted field cannot point the validator at a file already on the server. See [Security](security.md#file-uploads).
- Only single-file inputs are supported; a multi-file input (`name="photos[]"`) is not a file array.

### Testing and PSR-7 servers

Files that do not come from a PHP upload — in tests, or under a PSR-7 worker — fail the `is_uploaded_file()` check. Replace it with `Validator::setUploadedFileVerifier()`, which receives the `tmp_name` path; `null` restores the default:

```php
use Luany\Framework\Validation\Validator;

Validator::setUploadedFileVerifier(fn (string $path) => str_starts_with($path, sys_get_temp_dir()));

// in a test's tearDown()
Validator::setUploadedFileVerifier(null);
```

The verifier is static: it applies to every validation in the process until it is reset.

## Database rules

`unique` and `exists` query the database:

```php
'email'    => 'required|email|unique:users',            // column = field name
'username' => 'required|unique:users,login',            // column "login"
'tags'     => 'array|exists:tags,id',                   // every element must exist
'user_id'  => 'required|integer|exists:users,id',
```

When **updating** a record, ignore its own row with the third parameter (and a fourth for a primary key other than `id`):

```php
'email' => 'required|email|unique:users,email,' . $user->id,
'code'  => 'required|unique:products,code,' . $product->uuid . ',uuid',
```

The rules need a database connection. The Kernel wires them automatically to the container's `db` binding (a `PDO`), `connection` (an object with `getPdo()`) or `PDO::class` — the skeleton's `DatabaseServiceProvider` binds both `db` and `connection`. The connection is opened only when a rule runs.

Without one, these rules throw a `LogicException` rather than passing silently. In other setups, register a verifier yourself:

```php
use Luany\Framework\Validation\DatabasePresenceVerifier;
use Luany\Framework\Validation\Validator;

Validator::setPresenceVerifier(new DatabasePresenceVerifier($pdo));
```

Table and column names are checked (letters, digits, underscore, optional `schema.table`) and quoted; values are always bound.

## Custom rules

Custom rules, like built-in ones, are skipped for `null` and `''`.

### Closures

A closure receives the field, value and all data, and returns `true` / `null` to pass, `false` for a generic message, or a string as the error message:

```php
'username' => ['required', function (string $field, mixed $value, array $data) {
    return !in_array(strtolower($value), ['admin', 'root'], true) ?: "The {$field} is reserved.";
}],
```

### Rule objects

```php
use Luany\Framework\Validation\Rule;

class Uppercase implements Rule
{
    public function passes(string $field, mixed $value, array $data): bool
    {
        return is_string($value) && strtoupper($value) === $value;
    }

    public function message(string $field): string
    {
        return "The {$field} field must be uppercase.";
    }
}
```

```php
'code' => ['required', new Uppercase()],
```

### Named rules

`Validator::extend()` registers a rule usable in strings, with parameters. Register it in a provider's `boot()` method:

```php
Validator::extend(
    'slug',
    fn (string $field, mixed $value, array $params, array $data) => (bool) preg_match('/^[a-z0-9-]+$/', $value),
    'The :field may only contain lowercase letters, digits and dashes.'
);
```

```php
'handle' => 'required|slug|max:40',
```

`:field` in the message is replaced with the field name. `Validator::flushExtensions()` removes all named rules (useful in tests).

## Form request classes

`luany make:request StoreUserRequest` generates a class in `app/Http/Requests` that wraps the validator with `rules()`, `passes()`, `fails()`, `errors()` and `validated()`. See [Commands](cli.md).
