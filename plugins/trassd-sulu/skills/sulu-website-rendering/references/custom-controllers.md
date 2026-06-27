# Custom page controllers

Wire a custom controller to a page template through the template XML, then add
your own data on top of Sulu's resolved property data.

## 1. Point the template at your controller

```xml
<?xml version="1.0" ?>
<template xmlns="http://schemas.sulu.io/template/template"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://schemas.sulu.io/template/template http://schemas.sulu.io/template/template-1.0.xsd">

    ...
    <controller>App\Controller\Website\CustomController::indexAction</controller>
    ...
</template>
```

## 2. Extend the default page controller and add attributes

Override the attribute-building method, call `parent::...` so all normal
template/property data is still resolved, then add your keys. Pull in services
via the service-subscriber mechanism (`getSubscribedServices`).

> Version note: the FQCN of the base controller and the exact signature of the
> attribute method below come from the Sulu documentation
> (`DefaultController` / `getAttributes($attributes, StructureInterface $structure, $preview)`).
> This class was not resolvable in the Sulu 3.x source checkout used to ground
> the skill — confirm the base class and method against your installed Sulu
> before relying on it. If it is unavailable, render via
> `TemplateAttributeResolverInterface` instead (see
> `custom-route-base-template.md`).

```php
<?php

namespace App\Controller\Website;

use Sulu\Bundle\WebsiteBundle\Controller\DefaultController;
use Sulu\Component\Content\Compat\StructureInterface;

class CustomController extends DefaultController
{
    protected function getAttributes($attributes, StructureInterface $structure = null, $preview = false)
    {
        $attributes = parent::getAttributes($attributes, $structure, $preview);
        $attributes['myData'] = $this->get('my_custom_service')->getMyData();

        return $attributes;
    }

    public static function getSubscribedServices(): array
    {
        $subscribedServices = parent::getSubscribedServices();
        $subscribedServices['my_custom_service'] = 'your_service_id_or_class';

        return $subscribedServices;
    }
}
```

Rules:
- Always merge into the parent result; replacing it drops the template's
  property data.
- Declare every extra service in `getSubscribedServices()` (chained off
  `parent::getSubscribedServices()`), per the Symfony service-subscriber docs.
