# Collections

A `Contao\Model\Collection` wraps one or more models. It always holds at least
one model — when a query matches nothing, finders return `null`, never an empty
collection.

## Producing a collection

`findBy()` returns a collection for multi-row queries (column, value, optional
options array). Late static binding works the same as for single-row finders.

```php
$pages = PageModel::findBy('language', 'de');
$pages = PageModel::findByLanguage('de');   // late static binding
```

### Complex conditions

Pass arrays of WHERE clauses and a matching array of bound values:

```php
// language = "de" AND pid = 1
$pages = PageModel::findBy(['language = ?', 'pid = ?'], ['de', 1]);

// type = "store" AND id IN (...)
$foos = FoobarModel::findBy(
    ['type = ?', 'id IN (' . implode(',', array_map('\intval', $ids)) . ')'],
    ['store']
);
```

### findAll() and findMultipleByIds()

```php
$pages = PageModel::findAll();                  // every row of the table
$pages = PageModel::findMultipleByIds([1, 2, 3]); // a collection from given IDs
```

## Accessing data

Iterate with `foreach`; each item is a model instance.

```php
foreach ($pages as $page) {
    // $page is a PageModel — read or modify it
}
```

Pull columns out of every row without iterating manually:

```php
$titles = $pages->fetchEach('title'); // one column from each row
$rows   = $pages->fetchAll();         // all columns from each row
```
