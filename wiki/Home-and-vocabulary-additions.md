# Home page — sections to append

Two small additions for the **Home** page: a "start here" callout so readers land on the right
page first, and a note reconciling the "hexagonal" and "clean" vocabulary the wiki uses
interchangeably.

---

## Start here

New to this guide? Read in this order:

1. **[Code Structure](Code-Structure)** — the layout and the rules.
2. **[Example: The `orders` Domain, End to End](Example-Orders-Domain)** — the same rules as one
   concrete, compilable slice.
3. **[Cross-Domain Entity Dependencies in DDD](Cross-Domain-Entity-Dependencies-in-DDD)** — how
   domains depend on each other without coupling.
4. **[Inspirations](Inspirations)** — the source material, when you want the deeper *why*.

## Hexagonal, clean, DDD — how they fit here

This guide borrows from all three, so the vocabulary can look mixed. Here's the mapping used
throughout the wiki:

| Concept elsewhere | In this structure |
|---|---|
| Ports (hexagonal) | The `interface.go` files — `Orders`, `OrdersDatastore`, etc. |
| Adapters (hexagonal) | The implementations behind them — HTTP/gRPC handlers, `mysql`/`mongo` datastores |
| Entities / domain core (clean, DDD) | The `entity` packages, depended on by everything, depending on nothing |
| Aggregates & bounded contexts (DDD) | One `domain_<n>` per bounded context; reference across them by ID |

You don't need to pick a camp. The one rule that ties them together: **dependencies point inward,
toward `entity`; outer layers know about inner layers, never the reverse.**
