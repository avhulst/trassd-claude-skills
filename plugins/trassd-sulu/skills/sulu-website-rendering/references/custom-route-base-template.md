# Custom route reusing the page `base` template

To render a controller on a custom route (not bound to a page) using the same
`base.html.twig` as your pages, build the template attributes with
`TemplateAttributeResolverInterface::resolve()`. It merges your custom
parameters with the Sulu base attributes the `base` template expects.

```php
<?php

namespace App\Controller\Website;

use Sulu\Bundle\HttpCacheBundle\Cache\SuluHttpCache;
use Sulu\Bundle\WebsiteBundle\Resolver\TemplateAttributeResolverInterface;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\AsController;
use Symfony\Component\Routing\Annotation\Route;
use Twig\Environment;

#[AsController]
class StaticController
{
    #[Route("/custom", name: "app_custom")]
    public function indexAction(
        TemplateAttributeResolverInterface $resolver,
        Environment $twig
    ): Response {
        $response = new Response($twig->render(
            'static/custom.html.twig',
            $resolver->resolve([
                'customAttribute' => 'parameter',
            ])
        ));

        // Cached (public) response:
        $response->setPublic();
        $response->setMaxAge(240);
        $response->setSharedMaxAge(240);

        // Reverse-proxy TTL: how long the proxy should cache the page (seconds).
        $response->headers->set(SuluHttpCache::HEADER_REVERSE_PROXY_TTL, '604800'); // 1 week

        // Uncached private response (controller with user-specific data):
        // $response->setPrivate();
        // $response->setMaxAge(0);
        // $response->setSharedMaxAge(0);
        // $response->headers->addCacheControlDirective('no-cache', true);
        // $response->headers->addCacheControlDirective('must-revalidate', true);
        // $response->headers->addCacheControlDirective('no-store', true);

        return $response;
    }
}
```

The template can extend the page base:

```twig
{% extends "base.html.twig" %}

{% block content %}
    {{ customAttribute }}
{% endblock %}
```

`SuluHttpCache::HEADER_REVERSE_PROXY_TTL` resolves to the header
`X-Reverse-Proxy-TTL` (verified in Sulu 3.x source).
