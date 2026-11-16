# Validação

O Luany inclui um validador sem dependências. O helper `validate()` cobre o caso mais comum — validar, redirecionar para trás com os erros em caso de falha e devolver os dados limpos em caso de sucesso — e a classe `Validator` dá controlo total.

## Início rápido

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

    User::create($data);   // apenas os campos validados

    return redirect('/users');
}
```

Em caso de sucesso, `validate()` devolve **apenas os campos que têm regras**. Em caso de falha, lança uma `ValidationException`, que o Kernel converte numa resposta:

| Pedido | Resposta |
| --- | --- |
| Formulário no browser | Os erros e os dados antigos são guardados em flash na sessão, seguidos de um redirecionamento |
| JSON (`Accept: application/json` ou AJAX) | `422` com `{"message": "The given data was invalid.", "errors": {...}}` |

O redirecionamento vai para o terceiro argumento, quando indicado; caso contrário, para o `Referer` **se este apontar para o host atual**; caso contrário, para `/`:

```php
validate($request->body(), $rules, '/users/create');
```

Os dados antigos nunca incluem campos `password*`, `_token`, `csrf_token` nem `_method`. Adicione as suas próprias chaves no arranque:

```php
use Luany\Framework\Exceptions\ValidationException;

ValidationException::$dontFlash[] = 'credit_card';
```

## Mostrar os erros

Após o redirecionamento, os erros ficam na sessão sob `errors` (um array de mensagens por campo) e os dados antigos estão disponíveis através de `old()`:

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

Ambos são [dados flash](session.md#dados-flash): duram apenas até ao pedido seguinte.

Uma resposta de erro em JSON tem este aspeto:

```json
{
    "message": "The given data was invalid.",
    "errors": {
        "email": ["The email field must be a valid email address."]
    }
}
```

## Usar o Validator diretamente

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

| Método | Devolve |
| --- | --- |
| `Validator::make($data, $rules)` | Um validador já executado |
| `fails()` / `passes()` | `bool` |
| `errors()` | `['field' => ['message', ...]]` |
| `validated()` | Apenas os campos validados |

## Sintaxe das regras

As regras são uma string separada por barras verticais ou um array. Use um array para expressões regulares que contenham `|` e para objetos de regra ou closures personalizados:

```php
'code'  => 'required|string|max:10',
'phone' => ['required', 'regex:/^\+?[0-9 ]{7,15}$/'],
```

Um nome de regra desconhecido lança uma `InvalidArgumentException` — um erro de digitação nunca desativa a validação silenciosamente.

## Regras disponíveis

| Regra | Passa quando |
| --- | --- |
| `required` | Presente e diferente de `null`, `''` ou `[]` |
| `sometimes` | Flag: valida o campo apenas quando está presente nos dados |
| `nullable` | Flag: `null` ou `''` é aceite como `null`; as outras regras são ignoradas |
| `string` | Uma string |
| `email` | Um endereço de email válido |
| `integer` | Um int, ou uma string inteira como `"42"` ou `"-3"` (sem espaços à volta) |
| `numeric` | Um número ou uma string numérica |
| `boolean` | `true`, `false`, `1`, `0`, `"1"`, `"0"`, `"true"`, `"false"`, `"on"`, `"off"`, `"yes"`, `"no"` |
| `array` | Um array |
| `date` | Uma data que o `strtotime()` entenda |
| `after:{date\|field}` | Uma data posterior à data indicada ou ao valor de outro campo |
| `before:{date\|field}` | Uma data anterior à data ou ao campo indicados |
| `after_or_equal:{date\|field}` / `before_or_equal:{date\|field}` | Versões inclusivas |
| `min:{n}` / `max:{n}` | Tamanho de pelo menos / no máximo `n` (consulte [tamanhos](#tamanhos)) |
| `between:{min},{max}` | Tamanho dentro do intervalo, inclusive |
| `digits:{n}` | Exatamente `n` dígitos |
| `in:{a},{b},...` | Um dos valores listados (comparados como strings) |
| `not_in:{a},{b},...` | Nenhum dos valores listados |
| `regex:{pattern}` / `not_regex:{pattern}` | Corresponde / não corresponde ao padrão |
| `url` | Um URL absoluto `http` ou `https` |
| `confirmed` | Idêntico a `{field}_confirmation` |
| `same:{field}` / `different:{field}` | Igual a / diferente de outro campo |
| `unique:{table},{column},{ignore},{ignoreColumn}` | Nenhum registo tem este valor — consulte [regras de base de dados](#regras-de-base-de-dados) |
| `exists:{table},{column}` | Um registo tem este valor; nos arrays, todos os elementos têm de existir |

### Valores vazios

A maioria das regras **ignora** `null` e `''` — combine-as com `required` para tornar um campo obrigatório. `'email' => 'email'` aceita um campo vazio, mas rejeita `"abc"`.

### `nullable`, `sometimes` e `boolean`

Estas regras também moldam os dados validados:

| Dados | `'bio' => 'nullable\|string'` | `'bio' => 'sometimes\|string'` | `'active' => 'boolean'` |
| --- | --- | --- | --- |
| Campo ausente | não consta de `validated()` | não é validado, não consta de `validated()` | `false` |
| `''` | `null` | validado (`''` passa em `string`) | `false` |
| `"on"` / `"1"` | `"on"` / `"1"` | `"on"` / `"1"` | `true` |

Um campo `boolean` é sempre convertido para `bool`, e um campo ausente passa a `false` — exatamente o que uma checkbox não assinalada precisa. Adicione `sometimes` para deixar de fora um booleano ausente (em atualizações parciais).

### Tamanhos

`min`, `max` e `between` medem:

- **arrays** pelo número de elementos,
- **números e strings numéricas** pelo valor,
- **outras strings** pelo comprimento em caracteres.

> **Aviso** Uma string numérica é comparada pelo valor, não pelo comprimento: `"12345678"` passa em `min:8` porque 12345678 ≥ 8, e um nome como `"300"` falha em `max:255`. Use `digits:{n}` ou `regex` para códigos numéricos e PINs.

### Datas

`after`, `before` e as variantes `_or_equal` aceitam uma string de data ou o nome de outro campo:

```php
'starts_at' => 'required|date|after:today',
'ends_at'   => 'required|date|after:starts_at',
```

## Regras de base de dados

`unique` e `exists` consultam a base de dados:

```php
'email'    => 'required|email|unique:users',            // coluna = nome do campo
'username' => 'required|unique:users,login',            // coluna "login"
'tags'     => 'array|exists:tags,id',                   // todos os elementos têm de existir
'user_id'  => 'required|integer|exists:users,id',
```

Ao **atualizar** um registo, ignore a sua própria linha com o terceiro parâmetro (e com um quarto, se a chave primária não for `id`):

```php
'email' => 'required|email|unique:users,email,' . $user->id,
'code'  => 'required|unique:products,code,' . $product->uuid . ',uuid',
```

Estas regras precisam de uma ligação à base de dados. O Kernel liga-as automaticamente ao binding `db` do container (um `PDO`), a `connection` (um objeto com `getPdo()`) ou a `PDO::class` — o `DatabaseServiceProvider` do skeleton associa tanto `db` como `connection`. A ligação só é aberta quando uma regra é executada.

Sem ligação, estas regras lançam uma `LogicException` em vez de passarem silenciosamente. Noutras configurações, registe um verificador manualmente:

```php
use Luany\Framework\Validation\DatabasePresenceVerifier;
use Luany\Framework\Validation\Validator;

Validator::setPresenceVerifier(new DatabasePresenceVerifier($pdo));
```

Os nomes de tabelas e colunas são verificados (letras, dígitos, underscore, `schema.table` opcional) e colocados entre aspas; os valores são sempre passados como parâmetros (bound).

## Regras personalizadas

Tal como as regras incluídas, as regras personalizadas são ignoradas para `null` e `''`.

### Closures

Uma closure recebe o campo, o valor e todos os dados, e devolve `true` / `null` para passar, `false` para uma mensagem genérica, ou uma string como mensagem de erro:

```php
'username' => ['required', function (string $field, mixed $value, array $data) {
    return !in_array(strtolower($value), ['admin', 'root'], true) ?: "The {$field} is reserved.";
}],
```

### Objetos de regra

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

### Regras com nome

`Validator::extend()` regista uma regra que pode ser usada em strings, com parâmetros. Registe-a no método `boot()` de um provider:

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

O `:field` na mensagem é substituído pelo nome do campo. `Validator::flushExtensions()` remove todas as regras com nome (útil em testes).

## Classes de form request

`luany make:request StoreUserRequest` gera uma classe em `app/Http/Requests` que envolve o validador com `rules()`, `passes()`, `fails()`, `errors()` e `validated()`. Consulte [Comandos](cli.md).
