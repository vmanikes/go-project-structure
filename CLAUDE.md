# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A reference/template project structure for scalable Go applications (see README.md and the
[project wiki](https://github.com/vmanikes/go-project-structure/wiki)). Most files are currently
stubs (`package x` plus a guiding comment) — this is a skeleton meant to be filled in, not a
working service. `go.mod` declares only the module name (`go-project-structure`), with no Go
version pinned and no dependencies yet.

## Commands

- Build everything: `go build ./...`
- Run a single entrypoint: `go run ./src/cmd/command_1` (or `command_2`)
- There are no tests, Makefile, linter config, or CI config in the repo yet.

## Architecture

All code lives under `src/`. The structure separates transport, business logic, and data access
into distinct layers, each organized by `domain_<n>` (business domain) and, for handlers, by API
version (`v1`, ...):

- `cmd/<command>/main.go` — binary entrypoints, one directory per executable.
- `internal/config` — application configuration.
- `internal/entity/domain_<n>` — plain domain entities, shared across layers and decoupled from
  any transport or storage representation.
- `internal/handlers/{grpc,http,messaging}/domain_<n>/v<n>` — transport-layer handlers per
  protocol. Each has `handlers.go` (request handling) and `models.go` (wire-format request/response
  types), which convert to/from `entity` types.
- `internal/middlewares/{grpc,http,messaging}` — cross-cutting middleware, one package per
  transport.
- `internal/server/{grpc,http,messaging}` — server setup/bootstrapping, one package per transport.
- `internal/services/domain_<n>` — business logic for a domain. `interface.go` declares the
  domain's service interface (e.g. `Domain1`); `domain_<n>_impl.go` implements it.
- `internal/services/domain_<n>/datastore` — data-access abstraction for a domain.
  `interface.go` declares the datastore interface (e.g. `Domain1Datastore`); backend-specific
  subpackages (`mongo/`, `mysql/`) implement it, each with `types.go` for DB-specific types and
  a file that implements the interface and converts DB types to/from domain entities.

The consistent pattern across both `services` and `datastore` layers is: define the interface in
the parent package, implement it in a sibling/child package. When adding a new domain, mirror this
whole structure (`entity`, `handlers/*/domain_<n>`, `services/domain_<n>` with its `datastore`
subpackage) rather than introducing a different layout.

## Conventions and philosophy (from the wiki)

The layout follows hexagonal / clean architecture with DDD boundaries. Design intent worth
preserving when filling in the skeleton:

- **`go.mod` at the repo root, all code under `src/`.** The root `go.mod` exists so IDEs/LSP
  recognize the project; nothing else lives at root.
- **`internal` may only be imported from `cmd`.** Wire dependencies together in `cmd/*/main.go`.
- **No dumping-ground packages.** Do not add `util`, `pkg`, `common`, or similar. Code belongs
  in the layer/domain it serves.
- **Interface-driven layers.** Every service and datastore is defined by an interface so it can
  be mocked and swapped. `entity` types are the currency between layers; handler `models.go` and
  datastore `types.go` convert to/from entities so transport and DB representations never leak
  across boundaries.
- **Cross-domain dependencies reference by ID, never by embedded object.** Store `CustomerID`,
  not a `Customer`. This keeps aggregate roots independent — never save multiple aggregate roots
  in one transaction.
- **Domain services import only *interfaces* from other domains, never their concrete
  implementations** (prevents circular dependencies). A service coordinating multiple domains
  takes the other domains' interfaces as injected dependencies.
- **Prefer eventual consistency (domain events) over cross-aggregate transactions** for
  multi-domain operations.
