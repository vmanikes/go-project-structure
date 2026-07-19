# go-project-structure

An opinionated, batteries-included **project layout for scalable Go services** — microservices and
monoliths alike. It’s a skeleton you clone and fill in, built around hexagonal / clean architecture
with DDD boundaries so your app stays extensible, testable, and ready for change.

> 📖 **The full guide lives in the [Wiki](https://github.com/vmanikes/go-project-structure/wiki).**
> Start with [Code Structure](https://github.com/vmanikes/go-project-structure/wiki/Code-Structure),
> then the [end-to-end example](https://github.com/vmanikes/go-project-structure/wiki/Example-Orders-Domain-End-to-End).

## Use it as an agent skill

The conventions are packaged as an [Agent Skill](https://www.skills.sh/) so your AI coding agent
scaffolds and extends services in this layout automatically:

```bash
npx skills add vmanikes/go-project-structure
```

## Layout

```
├── go.mod                      # at the root, so IDEs/LSP recognize the project
└── src                         # all Go code lives here
    ├── cmd/<command>/main.go   # binary entrypoints — wiring only, no business logic
    └── internal                # importable only from cmd
        ├── config              # application configuration
        ├── entity/<domain>     # plain domain models, decoupled from transport & storage
        ├── handlers/{http,grpc,messaging}/<domain>/<version>
        │                       # transport handlers + wire models (convert to/from entity)
        ├── middlewares/{http,grpc,messaging}
        ├── server/{http,grpc,messaging}
        └── services/<domain>   # business logic behind an interface
            └── datastore       # data-access interface + per-DB impls (mysql, mongo, …)
```

## Principles

- **`go.mod` at root, code under `src/`.** `internal` is importable only from `cmd`; wire everything in `cmd/*/main.go`.
- **Interface-driven layers.** Every service and datastore is defined by an interface, so it’s mockable and swappable.
- **Entities are the currency between layers.** Handler `models.go` and datastore `types.go` convert to/from entities, so transport and DB representations never leak across boundaries.
- **No dumping grounds.** No `util`, `pkg`, or `common` — code belongs to the layer/domain it serves.
- **Cross-domain by ID, through interfaces.** Store `CustomerID`, not a `Customer`; import other domains’ interfaces, never their implementations. Prefer domain events over cross-aggregate transactions.

## How to use it

This repo is a reference skeleton — the files are stubs meant to be filled in, not a running
service. Copy the layout, then add each domain by mirroring the structure above: an
`entity/<domain>`, handlers under every transport it speaks, and a `services/<domain>` with its
`datastore` subpackage. The [Wiki](https://github.com/vmanikes/go-project-structure/wiki) explains
the reasoning behind every package.
