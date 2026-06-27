# Security expressions — full reference

> For complex authorization rules, prefer the **voter system**. Expressions are
> best for simple, inline combined conditions.

`isGranted()` and `#[IsGranted]` accept an
`Symfony\Component\ExpressionLanguage\Expression` in place of a role string.

## Attribute and method forms

```php
use Symfony\Component\ExpressionLanguage\Expression;
use Symfony\Component\Security\Http\Attribute\IsGranted;

class MyController extends AbstractController
{
    #[IsGranted(new Expression('is_granted("ROLE_ADMIN") or is_granted("ROLE_MANAGER")'))]
    public function show(): Response { /* ... */ }

    #[IsGranted(new Expression(
        '"ROLE_ADMIN" in role_names or (is_authenticated() and user.isSuperAdmin())'
    ))]
    public function edit(): Response { /* ... */ }
}
```

```php
$this->denyAccessUnlessGranted(new Expression(
    'is_granted("ROLE_ADMIN") or is_granted("ROLE_MANAGER")'
));
```

## Variables available in the expression

- **`user`** — the current `UserInterface`, or `null` if not authenticated.
- **`role_names`** — array of the user's role strings (includes roles from the
  role hierarchy; does **not** include the `IS_AUTHENTICATED_*` attributes).
- **`object`** — the object passed as the second argument to `isGranted()`.
- **`subject`** — same value as `object` (equivalent).
- **`token`** — the token object.
- **`trust_resolver`** — the `AuthenticationTrustResolverInterface` (usually you
  use the `is_*()` functions instead).

## Functions available in the expression

- **`is_authenticated()`** — `true` if logged in (fully or via remember-me).
- **`is_remember_me()`** — `true` *only* if authenticated via a remember-me
  cookie. Note this differs from checking `IS_AUTHENTICATED_REMEMBERED`.
- **`is_fully_authenticated()`** — `true` *only* if the user logged in during
  this session. Differs from `IS_AUTHENTICATED_FULLY`.
- **`is_granted()`** — checks a permission; optional second argument is the
  object to check on. Equivalent to the `isGranted()` method.

`is_remember_me()` / `is_fully_authenticated()` are similar to, but not the same
as, `IS_AUTHENTICATED_REMEMBERED` / `IS_AUTHENTICATED_FULLY`:

```php
$access1 = $authorizationChecker->isGranted('IS_AUTHENTICATED_REMEMBERED');
$access2 = $authorizationChecker->isGranted(new Expression(
    'is_remember_me() or is_fully_authenticated()'
));
// $access1 and $access2 yield the same result here
```

## Expression as the `#[IsGranted]` subject

The subject can itself be an expression, evaluated against the controller args:

```php
#[IsGranted(
    attribute: new Expression('user === subject'),
    subject: new Expression('args["post"].getAuthor()'),
)]
public function index(Post $post): Response { /* ... */ }
```

The subject can be an **array** where keys alias each expression's result:

```php
#[IsGranted(
    attribute: new Expression('user === subject["author"] and subject["post"].isPublished()'),
    subject: [
        'author' => new Expression('args["post"].getAuthor()'),
        'post',
    ],
)]
public function index(Post $post): Response { /* ... */ }
```

The subject can be the current request:

```php
#[IsGranted(attribute: '...', subject: new Expression('request'))]
public function index(): Response { /* ... */ }
```

Inside a subject expression you have two extra variables:

- **`request`** — the current `Request` object.
- **`args`** — array of controller arguments passed to the action.

## Closures (Symfony 7.3+, requires PHP 8.5)

`#[IsGranted]` also accepts a closure returning a boolean; the subject may be a
closure returning the values injected into it:

```php
use Symfony\Component\Security\Http\Attribute\IsGrantedContext;

#[IsGranted(
    static fn (IsGrantedContext $context, mixed $subject) =>
        $context->user === $subject['post']->getAuthor(),
    subject: static fn (array $args) => ['post' => $args['post']],
)]
public function index($post): Response { /* ... */ }
```
