# Cross-Domain Entity Dependencies — section to append

Paste this into the existing **Cross-Domain Entity Dependencies in DDD** page. The page states the
rules well; this shows the concrete failure the rules prevent, which makes the *why* land harder
than the principle alone.

---

## The failure this prevents: the import cycle

Suppose `orders` needs to confirm a customer exists. The tempting shortcut is to import the
`customers` **implementation** and, from there, the customer entity — and to have `customers`
reach back into `orders` for the customer's order history. Now:

```
services/orders   ──imports──▶  services/customers (impl)
services/customers ──imports──▶ services/orders   (impl)
```

Go refuses to compile this:

```
import cycle not allowed
package go-project-structure/src/internal/services/orders
	imports go-project-structure/src/internal/services/customers
	imports go-project-structure/src/internal/services/orders
```

There is no flag to turn this off — the cycle is a hard compile error. The rules on this page are
exactly what keep you out of it:

- **Reference by ID, not by object.** `Order.CustomerID string` means `orders` never needs to
  import the `customers` *entity* at all.
- **Import only interfaces from other domains.** `orders` depends on the `customers.Customers`
  interface, injected at construction. The interface has no dependency back on `orders`, so no
  cycle can form — even if the `customers` *implementation* later grows a dependency on `orders`.

```go
// services/orders/orders_impl.go
type service struct {
	store     datastore.OrdersDatastore
	customers customers.Customers // interface only — no cycle possible
}
```

If two domains truly need each other's data synchronously, that's a signal to reconsider the
boundary — or to move the coordination up into a higher-level service (or a domain event; see
below) rather than crossing implementations.
