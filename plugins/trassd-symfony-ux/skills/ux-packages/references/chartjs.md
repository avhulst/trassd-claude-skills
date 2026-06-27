# Chart.js (`symfony/ux-chartjs`)

Integrates the Chart.js library. Build the chart in PHP, render it in Twig.

## Build in PHP

Inject `ChartBuilderInterface` and create a `Chart`. Use the `Chart::TYPE_*`
constants (e.g. `Chart::TYPE_LINE`) to pick the chart type.

```php
use Symfony\UX\Chartjs\Builder\ChartBuilderInterface;
use Symfony\UX\Chartjs\Model\Chart;

public function index(ChartBuilderInterface $chartBuilder): Response
{
    $chart = $chartBuilder->createChart(Chart::TYPE_LINE);

    $chart->setData([
        'labels' => ['January', 'February', 'March'],
        'datasets' => [[
            'label' => 'My First dataset',
            'backgroundColor' => 'rgb(255, 99, 132)',
            'borderColor' => 'rgb(255, 99, 132)',
            'data' => [0, 10, 5],
        ]],
    ]);

    $chart->setOptions([
        'scales' => ['y' => ['suggestedMin' => 0, 'suggestedMax' => 100]],
    ]);

    return $this->render('home/index.html.twig', ['chart' => $chart]);
}
```

Everything passed to `setData()` / `setOptions()` is forwarded to Chart.js
verbatim — consult the Chart.js documentation for the option keys.

## Render in Twig

```twig
{{ render_chart(chart) }}

{# HTML attributes on the <canvas> go in the second argument #}
{{ render_chart(chart, {'class': 'my-chart'}) }}
```

## Plugins

Chart.js plugins are installed as JS packages and registered globally in
`assets/app.js` by listening for the `chartjs:init` event (fired once before the
first chart renders), then configured from PHP under `setOptions()` (e.g. a
`'plugins' => ['zoom' => [...]]` array). The PHP side only supplies config; the
plugin itself is registered in JS.

## Extending behavior

Attach a custom Stimulus controller through the render call and hook into the
`chartjs:pre-connect` (config not yet applied — mutate `event.detail.config`)
and `chartjs:connect` (`event.detail.chart` is the live instance) events:

```twig
{{ render_chart(chart, {'data-controller': 'mychart'}) }}
```

You do not write the base chart controller yourself — only the optional
extension controller.
