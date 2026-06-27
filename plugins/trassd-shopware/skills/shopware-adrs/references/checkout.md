# Checkout

## Payment flow (2021-10-01)

**Decision:** Custom payments implement one of two handlers. A **synchronous**
handler executes the payment immediately after the order is created, without
user interaction, and throws on error. An **asynchronous** handler is used when
the buyer must be redirected to an external payment gateway: it prepares the
redirect URL and later *finalizes* the return from the gateway. Either handler
may optionally support **pre-created payments** (the client prepares the payment
with the provider and passes a token; the handler must verify it before
charging) for headless flows. A failed payment enters the *after-order* loop
where the customer can pick another method and retry.

**Rule for extensions:** Pick synchronous vs asynchronous based on whether a
redirect is needed. Always validate externally created tokens against the
payment provider before completing the order. Don't assume a single attempt —
your handler can be re-entered through the after-order retry loop.

## Tax providers (2022-04-28)

**Decision:** To support jurisdictions with many tax rates, a tax provider runs
*after* the cart is fully calculated and can overwrite line-item, delivery and
total taxes. Providers are registered as `tax_provider` entities (with
`priority`, an availability rule, and a unique `providerIdentifier`) and
implemented as services tagged `shopware.tax.provider` implementing
`TaxProviderInterface`. They are called only when a customer is logged in and
the availability rule matches; highest priority runs first; throwing
`TaxProviderOutOfScopeException` (or returning nothing) falls through to the next
provider. App scripting can fill the provider via `TaxProviderHook`.

**Rule for extensions:** Implement `TaxProviderInterface::provideTax(Cart,
SalesChannelContext): TaxProviderStruct`, tag the service `shopware.tax.provider`,
and register a matching `tax_provider` entity whose `providerIdentifier` equals
the tag/app name. Fill only the parts of `TaxProviderStruct` you handle;
populating any value stops the provider chain.

```php
interface TaxProviderInterface
{
    public function provideTax(Cart $cart, SalesChannelContext $context): TaxProviderStruct;
}
```

## Checkout gateway (2024-04-01)

**Decision:** A `CheckoutGatewayInterface::process(CheckoutGatewayPayloadStruct):
CheckoutGatewayResponse` runs during checkout (confirm and edit-order pages) and
decides, from the cart and sales-channel context, which payment/shipping methods
are available and whether to block the cart with cart errors. The response is
built from a chain of gateway **commands** (`add-payment-method`,
`remove-payment-method`, `add-shipping-method`, `remove-shipping-method`,
`add-cart-error`). Apps participate by declaring a `gateways/checkout` endpoint
in `manifest.xml`; the `AppCheckoutGateway` calls each such app with the cart,
context and the technical names of available methods, and the app returns
commands. A `CheckoutGatewayCommandsCollectedEvent` lets plugins adjust commands
before execution.

**Rule for extensions:** Plugins implement `CheckoutGatewayInterface` (command
structure encouraged but optional) for logic driven by external systems
(ERP/PIM). Apps add the manifest endpoint and return command JSON, e.g.
removing a method or adding a blocking cart error.

```xml
<gateways>
    <checkout>https://example.com/checkout/gateway</checkout>
</gateways>
```

## Stock manipulation API (2023-05-15)

**Decision:** Stock handling moved to a single realtime `stock` field on the
product, managed through `AbstractStockStorage` with three methods: `load`
(augment products with stock data when read), `alter` (apply a list of
`StockAlteration` changesets as orders/items transition states), and `index`
(recalculate after product writes). `OrderStockSubscriber` translates order
lifecycle events into `alter` calls; the legacy `StockUpdater` and its filters
are deprecated. `available_stock` is kept only as a deprecated mirror of `stock`.
Behaviour can be disabled via `shopware.stock.enable_stock_management: false`.

**Rule for extensions:** To integrate custom inventory (e.g. an ERP or
multi-warehouse), decorate `AbstractStockStorage` and implement
`getDecorated()`/`load`/`alter`/`index` — don't write `available_stock` or
reimplement stock filters. To replace stock logic entirely, disable the core
stock management and provide your own storage and subscriber.

```php
abstract class AbstractStockStorage
{
    abstract public function getDecorated(): self;
    abstract public function load(StockLoadRequest $r, SalesChannelContext $c): StockDataCollection;
    abstract public function alter(array $changes, Context $c): void;   // list<StockAlteration>
    abstract public function index(array $productIds, Context $c): void;
}
```
</content>
