---
name: go-project-structure
description: Scaffold and extend Go backend services using a hexagonal/clean architecture with DDD boundaries. Use when creating a new Go service, adding a domain, or adding a handler/service/datastore layer to a Go backend — especially for HTTP/gRPC/messaging services that need to stay testable and swappable. Enforces interface-driven layers, entity-at-the-boundary conversions, and reference-by-ID between domains.
---

# Go Project Structure

An opinionated layout for scalable Go services (microservices and monoliths), based on hexagonal /
clean architecture with DDD boundaries. Follow it when scaffolding a service or adding to one so the
codebase stays extensible, testable, and swappable.

Full reasoning: https://github.com/vmanikes/go-project-structure/wiki

## The layout

```
├── go.mod                      # at the ROOT, so IDEs/LSP recognize the project
└── src                         # all Go code lives here
    ├── cmd/<command>/main.go   # binary entrypoints — wiring only, NO business logic
    └── internal                # importable ONLY from cmd
        ├── config              # application configuration
        ├── entity/<domain>     # plain domain models, decoupled from transport & storage
        ├── handlers/{http,grpc,messaging}/<domain>/<version>
        │                       # transport handlers + wire models (convert to/from entity)
        ├── middlewares/{http,grpc,messaging}
        ├── server/{http,grpc,messaging}
        └── services/<domain>   # business logic behind an interface
            └── datastore       # data-access interface + per-DB impls (mysql, mongo, …)
```

## Non-negotiable rules

1. **`go.mod` at repo root; all code under `src/`.** Nothing else lives at root.
2. **`internal` is importable only from `cmd`.** Wire concrete implementations together in `cmd/*/main.go` — that is the *only* place implementations meet.
3. **Interface-driven layers.** Every service and every datastore is defined by an interface (`interface.go` in the parent package) and implemented in a sibling/child package. This is what makes them mockable and swappable.
4. **Entities are the currency between layers.** Handler `models.go` (wire format) and datastore `types.go` (DB shape) convert *to/from* `entity` types. Transport and DB representations never leak across a boundary.
5. **No dumping grounds.** Never create `util`, `pkg`, or `common`. Code belongs to the layer/domain it serves.
6. **Cross-domain: reference by ID, depend on interfaces.** Store `CustomerID`, not an embedded `Customer`. A service that needs another domain takes that domain's *interface* as an injected dependency — never its implementation (that would create an import cycle Go rejects at compile time). Prefer domain events over transactions spanning aggregates.

## Adding a new domain (the core procedure)

Given a domain named e.g. `orders`, mirror this whole slice — do not invent a different layout:

1. **Entity** — `src/internal/entity/orders/entity.go`: the plain domain struct(s). Reference other domains by ID only (`CustomerID string`, not `*Customer`).

2. **Datastore interface** — `src/internal/services/orders/datastore/interface.go`:
   ```go
   package datastore
   type OrdersDatastore interface {
       Create(ctx context.Context, o orders.Order) error
       GetByID(ctx context.Context, id string) (orders.Order, error)
   }
   ```

3. **Datastore impl** — `src/internal/services/orders/datastore/mysql/` (and/or `mongo/`):
   - `types.go`: DB-shaped structs (rows/documents), which never leave this package.
   - `mysql.go`: implements `OrdersDatastore`, converting DB types ⇄ `entity.Order`.

4. **Service interface** — `src/internal/services/orders/interface.go`:
   ```go
   package orders
   type Orders interface {
       Place(ctx context.Context, customerID string, items []entity.Item) (entity.Order, error)
   }
   ```

5. **Service impl** — `src/internal/services/orders/orders_impl.go`: the business logic. Depends on the `datastore` interface and on any *other domain's* service interface (injected, never the impl).

6. **Handler(s)** — `src/internal/handlers/<transport>/orders/v1/`:
   - `models.go`: wire request/response types (JSON tags, proto types, etc.).
   - `handlers.go`: decode request → convert to entity args → call the service → convert entity → response. No business logic here.

7. **Wire it up** — in `cmd/<app>/main.go`: construct the datastore impl, inject it into the service, inject peer-domain services, hand the service to the handler, register routes. This is the composition root.

## Versioning

`v1`, `v2` version the **transport contract only**. Cut a `v2` for a breaking wire change; keep additive changes in `v1`. The `services` layer is never versioned — both handler versions call the same service interface, each doing its own conversion.

## Anti-patterns to reject

- A `util`/`pkg`/`common` package. Put the code in its layer/domain instead.
- An interface with reasoning like "we might swap it later" but zero conversion boundary — every service/datastore still gets its interface; that is the point.
- Embedding another domain's entity instead of holding its ID.
- Importing another domain's *implementation* package.
- Business logic in handlers, or DB/wire types leaking past their boundary.
- Anything at the repo root other than `go.mod` (and standard meta files like README/LICENSE).
