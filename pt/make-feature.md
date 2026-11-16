# make:feature

`luany make:feature` gera uma funcionalidade CRUD completa — modelo, migration, controller com validação, quatro vistas LTE e um ficheiro de rotas — com um único comando. Esta página aborda os tipos de campo, o código gerado e o modo interativo.

## Início rápido

```bash
luany make:feature Product name:string price:decimal description:text? category_id:foreign
luany migrate
luany dev
```

Abra `http://localhost:8000/products` para listar, criar, ver, editar e apagar produtos.

```bash
  →  Creating feature [Product]...

  ✓  Model created:      app/Models/Product.php
  ✓  Controller created: app/Controllers/ProductController.php
  ✓  Migration created:  database/migrations/2026_11_11_101500_create_products_table.php
  ✓  View created:       views/pages/products/index.lte
  ✓  View created:       views/pages/products/show.lte
  ✓  View created:       views/pages/products/create.lte
  ✓  View created:       views/pages/products/edit.lte
  ✓  Routes created:     routes/products.php
```

## Nomes

O nome da funcionalidade é convertido para StudlyCase (`product`, `Product` e `blog-post` são todos válidos) e tudo o resto é derivado dele:

| Funcionalidade | Modelo | Tabela | URL, ficheiro de rotas e pasta de vistas | Variáveis nas vistas |
|---|---|---|---|---|
| `Product` | `Product` | `products` | `products` | `$product`, `$products` |
| `Category` | `Category` | `categories` | `categories` | `$category`, `$categories` |
| `BlogPost` | `BlogPost` | `blog_posts` | `blog-posts` | `$blogpost`, `$blogposts` |

A pluralização segue um pequeno conjunto de regras do inglês: consoante + `y` → `ies`, `s`/`sh`/`ch`/`x`/`z` → `es`, caso contrário `s`. Os nomes compostos usam `snake_case` na tabela (`blog_posts`, também no modelo e nas regras de validação) e `kebab-case` nos URLs, no ficheiro de rotas e na pasta de vistas (`/blog-posts`, `routes/blog-posts.php`, `views/pages/blog-posts/`); as variáveis nas vistas ficam em minúsculas, sem separador (`$blogpost`). Antes da luany/cli 1.6, todos usavam `blogposts`.

## Campos

Os campos escrevem-se como `name:type`. Os nomes têm de estar em `snake_case` (letras minúsculas, dígitos e `_`); `id`, `created_at` e `updated_at` são reservados, porque todas as migrations já os definem. Nomes repetidos são rejeitados.

| Tipo | Coluna | Campo do formulário | Cast | Validação |
|---|---|---|---|---|
| `string` | `string()` | text | — | `required`, `string`, `max:255` |
| `text` | `text()` | textarea | — | `required`, `string` |
| `integer` | `integer()->default(0)` | number | `int` | `required`, `integer` |
| `decimal` | `decimal(10, 2)->default(0)` | number, `step="0.01"` | `decimal:2` | `required`, `numeric` |
| `boolean` | `boolean()->default(false)` | checkbox | `bool` | `boolean` |
| `email` | `string(150)->unique()` | email | — | `required`, `email`, `max:150`, `unique` |
| `date` | `date()->nullable()` | date | `date` | `nullable`, `date` |
| `datetime` | `dateTime()->nullable()` | datetime-local | `datetime` | `nullable`, `date` |
| `foreign` | `foreignId()->constrained()` | number, `min="1"` | `int` | `required`, `integer`, `exists` |

Um campo `boolean` dispensa a regra `required`: uma checkbox desmarcada não vem no pedido e é validada como `false`. Os campos `date` e `datetime` são sempre opcionais.

### Campos opcionais

Termine um campo com `?` para o tornar opcional:

```bash
luany make:feature Author name:string bio:text? website:string?
```

Um campo opcional recebe `->nullable()` na migration, `nullable` em vez de `required` nas regras de validação (`'bio' => ['nullable', 'string']`), e o respetivo campo no formulário de criação não é marcado como `required`. As colunas `integer` e `decimal` perdem o `default(0)`.

### Chaves estrangeiras

Um campo `foreign` tem de terminar em `_id`. A tabela referenciada é o prefixo no plural:

```bash
luany make:feature Product name:string category_id:foreign
```

| Gerado | Código |
|---|---|
| Migration | `$table->foreignId('category_id')->constrained('categories');` |
| Validação | `'category_id' => ['required', 'integer', 'exists:categories,id']` |
| Modelo | `public function category(): BelongsTo` que devolve `$this->belongsTo(Category::class, 'category_id')` |

A chave estrangeira precisa de que a tabela `categories` exista quando a migration é executada. Se não for encontrada nenhuma migration `create_categories_table`, o comando avisa:

```bash
  ⚠   No migration creates [categories] yet — create it before running migrate (foreign key).
```

Gere primeiro a funcionalidade-mãe (`luany make:feature Category name:string`), para que a sua migration tenha um timestamp anterior. A relação espera também um modelo `App\Models\Category`.

### Emails únicos

Um campo `email` recebe um índice único e uma regra `unique`. Em `update()`, a regra ignora o registo que está a ser editado, pelo que guardar um produto sem alterar o email é aceite:

```php
// store()
'email' => ['required', 'email', 'max:150', 'unique:authors,email'],
// update()
'email' => ['required', 'email', 'max:150', 'unique:authors,email,' . $id],
```

As regras `unique` e `exists` consultam a base de dados através do presence verifier que o framework configura a partir do binding `db` — consulte [Validação](validation.md).

## Ficheiros gerados

| Ficheiro | Conteúdo |
|---|---|
| `app/Models/{Model}.php` | `$table`, `$timestamps = true`, `$fillable` com todos os campos, `$casts`, relações `belongsTo()` |
| `database/migrations/{timestamp}_create_{table}_table.php` | Migration com o schema builder: `id()`, as colunas dos campos, `timestamps()`; `down()` apaga a tabela |
| `app/Controllers/{Model}Controller.php` | `index`, `show`, `create`, `store`, `edit`, `update`, `destroy` |
| `views/pages/{table}/index.lte` | Tabela com as ações View / Edit / Delete e um estado vazio |
| `views/pages/{table}/show.lte` | Cartão de detalhe (datas formatadas, booleanos como Yes/No) |
| `views/pages/{table}/create.lte` | Formulário com `@csrf` |
| `views/pages/{table}/edit.lte` | Formulário pré-preenchido com `@csrf` e `@method('PUT')` |
| `routes/{table}.php` | `Route::resource('{table}', {Model}Controller::class);` |

Os ficheiros existentes nunca são substituídos: cada um que já exista é ignorado com um aviso, e os restantes são gerados na mesma.

### Modelo

```php
<?php

namespace App\Models;

use Luany\Database\Relations\BelongsTo;
use Luany\Database\Model;

class Product extends Model
{
    protected string $table = 'products';

    /** created_at / updated_at are written by the model. */
    protected bool $timestamps = true;

    protected array $fillable = [
        'name',
        'price',
        'description',
        'category_id',
    ];

    protected array $hidden = [];

    protected array $casts = [
        'price' => 'decimal:2',
        'category_id' => 'int',
    ];

    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class, 'category_id');
    }
}
```

Consulte [Modelos](models.md) para casts e timestamps, e [Relações](relationships.md) para `belongsTo()`.

### Migration

A migration usa o schema builder portável, pelo que funciona em SQLite, MySQL e PostgreSQL:

```php
public function up(\PDO $pdo): void
{
    $this->schema($pdo)->create('products', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->decimal('price', 10, 2)->default(0);
        $table->text('description')->nullable();
        $table->foreignId('category_id')->constrained('categories');
        $table->timestamps();
    });
}
```

### Controller

Cada ação recebe o parâmetro de rota `$id` pelo nome. Registos inexistentes chamam `abort(404)`, e `store()` / `update()` validam o corpo do pedido antes de escrever:

```php
public function store(Request $request): Response
{
    $data = validate($request->body(), [
        'name' => ['required', 'string', 'max:255'],
        'price' => ['required', 'numeric'],
        'description' => ['nullable', 'string'],
        'category_id' => ['required', 'integer', 'exists:categories,id'],
    ], '/products/create');

    Product::create($data);
    flash('success', 'Product created successfully.');

    return redirect('/products');
}
```

Em caso de falha, `validate()` guarda os erros e os dados antigos como mensagens flash e redireciona de volta para o formulário. Só os campos validados chegam a `create()` e `update()`.

### Rotas

O ficheiro de rotas é carregado automaticamente: o Kernel do framework carrega primeiro `routes/http.php` e depois todos os outros ficheiros `routes/*.php` por ordem alfabética.

| Método | URI | Ação |
|---|---|---|
| GET | `/products` | `index` |
| GET | `/products/create` | `create` |
| POST | `/products` | `store` |
| GET | `/products/{id}` | `show` |
| GET | `/products/{id}/edit` | `edit` |
| PUT / PATCH | `/products/{id}` | `update` |
| DELETE | `/products/{id}` | `destroy` |

Os formulários HTML enviam `PUT` e `DELETE` através do campo `_method` escrito por `@method`. Consulte [Rotas](routing.md).

## Modo interativo

Execute o comando apenas com um nome para definir os campos passo a passo:

```bash
luany make:feature Product
```

```bash
  Tip: luany make:feature Product title:string price:decimal bio:text?

  Add fields? [Y/n]:

  Enter a name (title) or name:type (price:decimal). End with ? for optional (bio:text?).
  Leave empty to finish.

  Field: name
  Type of [name]
   ›  1  string     short text (255)
      2  text       long text
      3  integer    whole number
      4  decimal    money / decimal (10,2)
      5  boolean    yes / no (checkbox)
      6  email      unique email address
      7  date       date
      8  datetime   date and time
      9  foreign    link to another table (name must end in _id)
  Choose 1-9 or a name [string]:
  ✓  name string
  Field: price:decimal
  ✓  price decimal
  Field: category_id
  ...
```

- Introduza apenas um nome para escolher o tipo no menu (número ou nome do tipo). O predefinido é `string`, ou `foreign` para nomes terminados em `_id`.
- Introduza `name:type` ou `name:type?` para saltar o menu.
- Um nome ou tipo inválido mostra um erro e volta a perguntar — nada se perde.
- Prima Enter numa linha vazia (ou escreva `done`) para terminar.

É então apresentado um resumo do modelo, da tabela e dos campos, seguido de `Generate model, migration, controller, views and routes? [Y/n]`. Responder `n` cancela sem escrever nenhum ficheiro. Responder `n` a `Add fields?` gera a estrutura sem campos.

> **Nota** Os campos indicados na linha de comandos têm sempre de incluir um tipo (`title:string`); a forma simples `title` só é aceite no modo interativo.

## Escolha do layout

As vistas geradas estendem um layout de `views/layouts/`:

| Layouts encontrados | Comportamento |
|---|---|
| Nenhum | É usado `layouts.main` |
| Um | É usado esse layout (`Using layout: layouts.app`) |
| Vários | Um menu numerado, com `main` como predefinido quando existe |

O layout é escolhido antes dos campos, mesmo quando os campos são indicados na linha de comandos. As vistas usam as variáveis CSS do `public/assets/css/app.css` do skeleton; ajuste os blocos `@style` se o seu layout usar estilos diferentes.

## Próximos passos

- Execute `luany migrate` para criar a tabela.
- Verifique as rotas com `luany route:list`.
- Valide os templates gerados com [view:check](view-check.md).
