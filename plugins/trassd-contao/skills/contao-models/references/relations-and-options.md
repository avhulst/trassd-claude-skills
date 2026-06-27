# Query options and relations

## Options array

The last argument to a finder is an options array merged into the SQL query:

| Option   | Effect                                  | SQL         | Example                  |
|----------|-----------------------------------------|-------------|--------------------------|
| `limit`  | Limit total records                     | `LIMIT`     | `3`                      |
| `offset` | Skip the first n records                | `OFFSET`    | `10`                     |
| `order`  | Sort by a field                         | `ORDER BY`  | `'id DESC'`              |
| `return` | Force return value                      | —           | `'Model'` / `'Collection'` |
| `eager`  | Load related records (hasOne/belongsTo) | `LEFT JOIN` | `true`                   |
| `having` | HAVING clause                           | `HAVING`    | `'id = 1'`               |

```php
$options = ['limit' => 5, 'offset' => 10];

$pages = PageModel::findBy('pid', 1, $options);  // explicit column/value
$pages = PageModel::findByPid(1, $options);      // late static binding
```

## Eager loading relations

If a DCA field defines a relation of type `hasOne` or `belongsTo`, setting
`'eager' => true` loads the related records in the same database call using a
join. Joined columns are prefixed with the source column name followed by a
double underscore (`<column>__<field>`), which lets you filter on them via
`having`.

```php
// tl_article DCA
'author' => [
    'foreignKey' => 'tl_user.name',
    'relation' => ['type' => 'hasOne', 'load' => 'lazy'],
],

// application code
use Contao\ArticleModel;

$articles = ArticleModel::findBy('tl_article.published = ?', true, [
    'return' => 'Array',
    'eager'  => true,
    'having' => "author__username = 'k.jones' AND author__disable != '1'",
]);

$article = $articles[0];                  // an ArticleModel
$author  = $article->getRelated('author'); // a UserModel
```

Use `$model->getRelated('fieldName')` to access a related model instance for a
relation field.
