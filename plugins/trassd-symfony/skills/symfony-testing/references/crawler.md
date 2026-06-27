# DOM crawler, links and forms

A `Crawler` is returned by every `$client->request(...)`. It traverses HTML/XML
like jQuery; each method returns a new `Crawler`, so calls chain.

## Traversing

```php
$newCrawler = $crawler->filter('input[type=submit]')
    ->last()
    ->ancestors()
    ->first();
```

| Method | Selects |
| --- | --- |
| `filter('h1.title')` | nodes matching a CSS selector |
| `filterXpath('//h1')` | nodes matching an XPath expression |
| `eq(1)` | node at index (`0` is first) |
| `first()` / `last()` | first / last node |
| `siblings()` | sibling nodes (same parent, excluding current) |
| `nextAll()` / `previousAll()` | following / preceding siblings |
| `ancestors()` | parents up to `<html>` |
| `children()` | direct child nodes |
| `reduce($cb)` | keep nodes where `$cb` returns `true` |

`count($crawler)` returns the number of nodes.

```php
$crawler
    ->filter('h1')
    ->reduce(function ($node, int $i): bool {
        return (bool) $node->attr('class');
    })
    ->first();
```

## Extracting information

```php
$crawler->attr('class');                  // attribute of first node
$crawler->text();                         // node value of first node
$crawler->text('Default text content');   // default if node missing
$crawler->text(null, true);               // normalize whitespace

// array of [_text, href] for each node (_text is the node value)
$info = $crawler->extract(['_text', 'href']);

// run a lambda per node, collect results
$data = $crawler->each(fn ($node, int $i) => $node->attr('href'));
```

## Clicking links

```php
$client->clickLink('Click here'); // first link with that text (or image alt)

// or get the Link object (getMethod(), getUri(), ...)
$link = $crawler->selectLink('Click here')->link();
$client->click($link);
```

## Submitting forms

Select a button, not a form (a form can have several buttons):

```php
$crawler = $client->submitForm('Add comment', [
    'comment_form[content]' => '...',
]);
```

First arg is the text/`id`/`name` of any `<button>` or `<input type="submit">`;
second arg overrides field values.

For full control, get the `Form` object (`getUri()`, `getValues()`,
`getFiles()`):

```php
$buttonCrawlerNode = $crawler->selectButton('submit');
$form = $buttonCrawlerNode->form();

$form['my_form[name]'] = 'Fabien';
$form['my_form[subject]'] = 'Symfony rocks!';
$client->submit($form);

// or pass values while submitting
$client->submit($form, [
    'my_form[name]'    => 'Fabien',
    'my_form[subject]' => 'Symfony rocks!',
]);
```

Filling specific field types:

```php
$form['my_form[country]']->select('France'); // option / radio
$form['my_form[like_symfony]']->tick();      // checkbox
$form['my_form[photo]']->upload('/path/to/lucas.jpg');
```

`getValues()` / `getFiles()` (and the `getPhpValues()` / `getPhpFiles()`
variants) return what will be submitted. `submit()` and `submitForm()` accept
extra args for server params and HTTP headers.
