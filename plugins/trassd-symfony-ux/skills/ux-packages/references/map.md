# Map (`symfony/ux-map`)

Renders interactive maps. Build a `Map` object in PHP, render with `ux_map()`.
**The Twig function is `ux_map()` — not `render_map()`.**

## Pick a renderer

UX Map is renderer-agnostic; you must install one renderer bridge and select it
via the `UX_MAP_DSN`:

- **Google Maps:** `composer require symfony/ux-google-map`,
  `UX_MAP_DSN=google://GOOGLE_MAPS_API_KEY@default`
- **Leaflet:** `composer require symfony/ux-leaflet-map`,
  `UX_MAP_DSN=leaflet://default`

```yaml
# config/packages/ux_map.yaml
ux_map:
    renderer: '%env(resolve:default::UX_MAP_DSN)%'
```

## Build a map in PHP

```php
use Symfony\UX\Map\Map;
use Symfony\UX\Map\Point;
use Symfony\UX\Map\Marker;
use Symfony\UX\Map\InfoWindow;

$map = (new Map())
    ->center(new Point(46.903354, 1.888334))
    ->zoom(6)
    // or: ->fitBoundsToMarkers()
    ->minZoom(3)->maxZoom(10)   // ensure minZoom <= zoom <= maxZoom
    ->addMarker(new Marker(
        position: new Point(48.8566, 2.3522),
        title: 'Paris',
        infoWindow: new InfoWindow(
            headerContent: '<b>Paris</b>',
            content: 'The capital of France.',
        ),
    ));
```

Markers accept an `Icon` (`Icon::ux('fa:map-marker')` — needs `symfony/ux-icons`,
`Icon::url(...)`, or `Icon::svg(...)`). You can also add `Polygon`, `Polyline`,
`Circle`, and `Rectangle` shapes (each via `add*()`), remove them with
`remove*($instanceOrId)`, or clear all with `removeAll*()`. Any element and the
map itself accept an `extra: [...]` array forwarded to the JS controller.

## Render in Twig

The map must have a defined height.

```twig
{{ ux_map(my_map, { style: 'height: 300px' }) }}
{{ ux_map(my_map, { style: 'height: 300px', id: 'events-map', class: 'mb-3' }) }}
```

`ux_map()` can also build a map inline (no PHP object) from `center`, `zoom`,
`markers`, `fitBoundsToMarkers`, and `attributes` arguments.

Component syntax (`<twig:ux:map />`) is also available and needs
`symfony/ux-twig-component`.

## Customizing / low-level options

Attach a Stimulus controller via `{'data-controller': 'mymap'}` and listen for
events such as `ux:map:pre-connect`, `ux:map:connect`, and the per-element
`ux:map:*:before-create` / `*:after-create` events. In `*:before-create` you can
mutate `event.detail.definition`, and set `event.detail.definition.bridgeOptions`
to pass renderer-native options straight to the underlying Google Maps / Leaflet
object.

## Live Components

Use `ComponentWithMapTrait` and implement `instantiateMap(): Map`. Inside
`#[LiveAction]` methods, get the map with `$this->getMap()` and mutate it
(`->center(...)`, `->zoom(...)`, `->addMarker(...)`, etc.). Disable
`fitBoundsToMarkers(false)` if you don't want auto-fit after adding elements.

## Clustering

Group nearby points with `GridClusteringAlgorithm` or `MortonClusteringAlgorithm`:
call `->cluster($points, zoom: 5.0)`, then iterate clusters
(`getCenter()`, `getPoints()`, `count()`) and add cluster markers.
