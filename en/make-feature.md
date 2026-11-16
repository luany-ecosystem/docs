# make:feature

`luany make:feature` scaffolds a complete CRUD feature — model, migration, controller with validation, four LTE views and a routes file — from a single command. This page covers field types, the generated code and the interactive mode.

## Quick start

```bash
luany make:feature Product name:string price:decimal description:text? category_id:foreign
luany migrate
luany dev
```

Open `http://localhost:8000/products` to list, create, show, edit and delete products.

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

## Naming

The feature name is converted to StudlyCase (`product`, `Product` and `blog-post` are all valid) and everything else is derived from it:

| Feature | Model | Table, URL and view folder | View variables |
|---|---|---|---|
| `Product` | `Product` | `products` | `$product`, `$products` |
| `Category` | `Category` | `categories` | `$category`, `$categories` |
| `BlogPost` | `BlogPost` | `blogposts` | `$blogpost`, `$blogposts` |

Pluralization is a small English rule set: consonant + `y` → `ies`, `s`/`sh`/`ch`/`x`/`z` → `es`, otherwise `s`. Multi-word names are lower-cased without a separator (`blogposts`).

## Fields

Fields are written as `name:type`. Names must be `snake_case` (lowercase letters, digits and `_`); `id`, `created_at` and `updated_at` are reserved because every migration already defines them. Duplicate names are rejected.

| Type | Column | Form input | Cast | Validation |
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

A `boolean` field needs no `required` rule: an unchecked checkbox is absent from the request and validates as `false`. `date` and `datetime` fields are always optional.

### Optional fields

End a field with `?` to make it optional:

```bash
luany make:feature Author name:string bio:text? website:string?
```

An optional field gets `->nullable()` in the migration, `nullable` instead of `required` in the validation rules (`'bio' => ['nullable', 'string']`), and its create-form input is not marked `required`. `integer` and `decimal` columns lose their `default(0)`.

### Foreign keys

A `foreign` field must end in `_id`. The referenced table is the pluralized prefix:

```bash
luany make:feature Product name:string category_id:foreign
```

| Generated | Code |
|---|---|
| Migration | `$table->foreignId('category_id')->constrained('categories');` |
| Validation | `'category_id' => ['required', 'integer', 'exists:categories,id']` |
| Model | `public function category(): BelongsTo` returning `$this->belongsTo(Category::class, 'category_id')` |

The foreign key needs the `categories` table to exist when the migration runs. If no `create_categories_table` migration is found, the command warns you:

```bash
  ⚠   No migration creates [categories] yet — create it before running migrate (foreign key).
```

Generate the parent feature first (`luany make:feature Category name:string`) so its migration has an earlier timestamp. The relation also expects an `App\Models\Category` model.

### Unique emails

An `email` field gets a unique index and a `unique` rule. In `update()` the rule ignores the record being edited, so saving a product without changing its email passes:

```php
// store()
'email' => ['required', 'email', 'max:150', 'unique:authors,email'],
// update()
'email' => ['required', 'email', 'max:150', 'unique:authors,email,' . $id],
```

The `unique` and `exists` rules query the database through the presence verifier the framework configures from the `db` binding — see [Validation](validation.md).

## Generated files

| File | Contents |
|---|---|
| `app/Models/{Model}.php` | `$table`, `$timestamps = true`, `$fillable` with every field, `$casts`, `belongsTo()` relations |
| `database/migrations/{timestamp}_create_{table}_table.php` | Schema-builder migration: `id()`, the field columns, `timestamps()`; `down()` drops the table |
| `app/Controllers/{Model}Controller.php` | `index`, `show`, `create`, `store`, `edit`, `update`, `destroy` |
| `views/pages/{table}/index.lte` | Table with View / Edit / Delete actions and an empty state |
| `views/pages/{table}/show.lte` | Detail card (dates formatted, booleans as Yes/No) |
| `views/pages/{table}/create.lte` | Form with `@csrf` |
| `views/pages/{table}/edit.lte` | Pre-filled form with `@csrf` and `@method('PUT')` |
| `routes/{table}.php` | `Route::resource('{table}', {Model}Controller::class);` |

Existing files are never overwritten: each one that already exists is skipped with a warning, and the rest are still generated.

### Model

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

See [Models](models.md) for casts and timestamps, and [Relationships](relationships.md) for `belongsTo()`.

### Migration

The migration uses the portable schema builder, so it runs on SQLite, MySQL and PostgreSQL:

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

Each action receives the route parameter `$id` by name. Missing records call `abort(404)`, and `store()` / `update()` validate the request body before writing:

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

On failure, `validate()` flashes the errors and old input and redirects back to the form. Only validated fields reach `create()` and `update()`.

### Routes

The routes file is loaded automatically: the framework Kernel loads `routes/http.php` first and then every other `routes/*.php` file alphabetically.

| Method | URI | Action |
|---|---|---|
| GET | `/products` | `index` |
| GET | `/products/create` | `create` |
| POST | `/products` | `store` |
| GET | `/products/{id}` | `show` |
| GET | `/products/{id}/edit` | `edit` |
| PUT / PATCH | `/products/{id}` | `update` |
| DELETE | `/products/{id}` | `destroy` |

HTML forms send `PUT` and `DELETE` through the `_method` field written by `@method`. See [Routing](routing.md).

## Interactive mode

Run the command with only a name to define fields step by step:

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

- Enter a bare name to pick the type from the menu (number or type name). The default is `string`, or `foreign` for names ending in `_id`.
- Enter `name:type` or `name:type?` to skip the menu.
- An invalid name or type shows an error and asks again — nothing is lost.
- Press Enter on an empty line (or type `done`) to finish.

A summary of the model, table and fields is then shown, followed by `Generate model, migration, controller, views and routes? [Y/n]`. Answering `n` aborts without writing any file. Answering `n` to `Add fields?` generates the structure with no fields.

> **Note** Inline fields must always include a type (`title:string`); the bare `title` form is accepted only in interactive mode.

## Layout selection

Generated views extend a layout from `views/layouts/`:

| Layouts found | Behaviour |
|---|---|
| None | `layouts.main` is used |
| One | That layout is used (`Using layout: layouts.app`) |
| Several | A numbered menu, defaulting to `main` when it exists |

The layout is chosen before the fields, also when fields are given inline. The views use the CSS variables of the skeleton's `public/assets/css/app.css`; adjust the `@style` blocks if your layout uses different styles.

## Next steps

- Run `luany migrate` to create the table.
- Check the routes with `luany route:list`.
- Validate the generated templates with [view:check](view-check.md).
