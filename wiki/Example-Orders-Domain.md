# Example: The `orders` Domain, End to End

The rest of the wiki describes the layout in the abstract. This page traces **one real domain**
through every layer so the pattern is concrete. Follow the same slice whenever you add a domain.

The request: `POST /v1/orders` creates an order for a customer. `orders` depends on the
`customers` domain — but only by ID and only through an interface (see
[Cross-Domain Entity Dependencies in DDD](Cross-Domain-Entity-Dependencies-in-DDD)).

```
HTTP request
  └─ handlers/http/orders/v1  (wire ⇄ entity)
       └─ services/orders     (business logic, interface + impl)
            ├─ services/customers (interface only — verify customer exists)
            └─ services/orders/datastore  (interface)
                 └─ .../datastore/mysql   (DB types ⇄ entity)
```

## 1. Entity — the currency between layers

`src/internal/entity/orders/entity.go`

```go
package orders

import "time"

// Order is the domain model. Note CustomerID, not an embedded Customer:
// aggregates reference each other by ID only.
type Order struct {
	ID         string
	CustomerID string
	Items      []Item
	Total      int64 // cents
	CreatedAt  time.Time
}

type Item struct {
	ProductID string
	Quantity  int
	UnitPrice int64
}
```

## 2. Datastore — interface in the parent, impl in the child

`src/internal/services/orders/datastore/interface.go`

```go
package datastore

import (
	"context"

	"go-project-structure/src/internal/entity/orders"
)

type OrdersDatastore interface {
	Create(ctx context.Context, order orders.Order) error
	GetByID(ctx context.Context, id string) (orders.Order, error)
}
```

`src/internal/services/orders/datastore/mysql/types.go` — DB-shaped types stay here and never
leak out:

```go
package mysql

import "time"

type orderRow struct {
	ID         string    `db:"id"`
	CustomerID string    `db:"customer_id"`
	Total      int64     `db:"total_cents"`
	CreatedAt  time.Time `db:"created_at"`
}
```

`src/internal/services/orders/datastore/mysql/mysql.go` — implements the interface and converts
`orderRow` ⇄ `orders.Order`:

```go
package mysql

import (
	"context"
	"database/sql"

	"go-project-structure/src/internal/entity/orders"
)

type Store struct{ db *sql.DB }

func New(db *sql.DB) *Store { return &Store{db: db} }

func (s *Store) Create(ctx context.Context, o orders.Order) error {
	row := orderRow{ID: o.ID, CustomerID: o.CustomerID, Total: o.Total, CreatedAt: o.CreatedAt}
	_, err := s.db.ExecContext(ctx,
		`INSERT INTO orders (id, customer_id, total_cents, created_at) VALUES (?, ?, ?, ?)`,
		row.ID, row.CustomerID, row.Total, row.CreatedAt)
	return err
}

func (s *Store) GetByID(ctx context.Context, id string) (orders.Order, error) {
	// ... scan into orderRow, then map to orders.Order
	return orders.Order{}, nil
}
```

Swapping to Mongo means adding `datastore/mongo/` that implements the same `OrdersDatastore`
interface — no service or handler changes.

## 3. Service — business logic behind an interface

`src/internal/services/orders/interface.go`

```go
package orders

import (
	"context"

	entity "go-project-structure/src/internal/entity/orders"
)

type Orders interface {
	Place(ctx context.Context, customerID string, items []entity.Item) (entity.Order, error)
}
```

`src/internal/services/orders/orders_impl.go` — depends on the datastore interface **and** on the
`customers` service *interface* (never its impl), so the two domains stay decoupled and can't form
an import cycle:

```go
package orders

import (
	"context"
	"time"

	entity "go-project-structure/src/internal/entity/orders"
	"go-project-structure/src/internal/services/customers"
	"go-project-structure/src/internal/services/orders/datastore"
)

type service struct {
	store     datastore.OrdersDatastore
	customers customers.Customers // interface from the other domain
}

func New(store datastore.OrdersDatastore, c customers.Customers) Orders {
	return &service{store: store, customers: c}
}

func (s *service) Place(ctx context.Context, customerID string, items []entity.Item) (entity.Order, error) {
	if _, err := s.customers.GetByID(ctx, customerID); err != nil {
		return entity.Order{}, err // cross-domain check, by ID
	}
	order := entity.Order{
		ID:         newID(),
		CustomerID: customerID,
		Items:      items,
		Total:      sum(items),
		CreatedAt:  time.Now(),
	}
	if err := s.store.Create(ctx, order); err != nil {
		return entity.Order{}, err
	}
	return order, nil
}
```

## 4. Handler — wire format ⇄ entity, one version deep

`src/internal/handlers/http/orders/v1/models.go` — the JSON contract lives here, separate from the
entity, so the API can evolve independently of the domain model:

```go
package v1

type createOrderRequest struct {
	CustomerID string     `json:"customer_id"`
	Items      []itemJSON `json:"items"`
}

type itemJSON struct {
	ProductID string `json:"product_id"`
	Quantity  int    `json:"quantity"`
	UnitPrice int64  `json:"unit_price"`
}

type orderResponse struct {
	ID    string `json:"id"`
	Total int64  `json:"total"`
}
```

`src/internal/handlers/http/orders/v1/handlers.go` — converts request → entity args, calls the
service, converts the entity → response. No business logic here:

```go
package v1

import (
	"encoding/json"
	"net/http"

	entity "go-project-structure/src/internal/entity/orders"
	"go-project-structure/src/internal/services/orders"
)

type Handler struct{ svc orders.Orders }

func NewHandler(svc orders.Orders) *Handler { return &Handler{svc: svc} }

func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
	var req createOrderRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}
	items := make([]entity.Item, len(req.Items))
	for i, it := range req.Items {
		items[i] = entity.Item{ProductID: it.ProductID, Quantity: it.Quantity, UnitPrice: it.UnitPrice}
	}
	order, err := h.svc.Place(r.Context(), req.CustomerID, items)
	if err != nil {
		http.Error(w, "could not place order", http.StatusInternalServerError)
		return
	}
	_ = json.NewEncoder(w).Encode(orderResponse{ID: order.ID, Total: order.Total})
}
```

## 5. Wiring — the only place implementations meet

`src/cmd/app/main.go` is the composition root. It's the one spot that imports concrete
implementations and injects them; every layer above talks to interfaces:

```go
func main() {
	db := openMySQL(cfg)

	customerSvc := customers.New(customerstore.New(db))
	orderSvc := orders.New(orderstore.New(db), customerSvc)

	h := ordersv1.NewHandler(orderSvc)
	// register h.Create on the router, start the server...
}
```

## Why each conversion exists

| Boundary | Types on each side | Why they're separate |
|---|---|---|
| Handler ⇄ service | `models.go` (JSON) vs `entity` | API contract can change without touching the domain, and vice-versa |
| Service ⇄ datastore | `entity` vs `types.go` (rows/docs) | DB schema and domain model evolve independently; swap DB with no service change |
| Domain ⇄ domain | ID + interface only | No import cycles, aggregates stay independent |
