# Code Structure — sections to append

Paste these into the existing **Code Structure** page. They fill the gaps a first-time reader
hits: how config is loaded, when to cut a new API version, and where tests/mocks live.

---

## Loading configuration

`config` holds the shape of your settings (HTTP, databases, third-party services), but something
has to populate it. Do that **once, in `cmd/*/main.go`**, before wiring anything else — never read
`os.Getenv` scattered through services.

- Load from environment variables (12-factor) or a file, with env taking precedence.
- Validate on startup and fail fast: a missing DB DSN should stop the process, not surface as a
  nil-pointer panic on the first request.
- Pass the resulting typed config value (or the sub-structs each layer needs) down through
  constructors. Layers receive config as arguments; they do not reach out for it.

```go
func main() {
	cfg, err := config.Load() // env + optional file, validated here
	if err != nil {
		log.Fatalf("config: %v", err)
	}
	db := openMySQL(cfg.MySQL)
	// ...
}
```

## Versioning handlers (`{type}/{domain}/{version}`)

The `v1`, `v2` directories version the **transport contract**, not the business logic.

- **Cut a `v2` only for a breaking wire change** — a renamed/removed field, a changed type, new
  required input. Additive, backward-compatible changes stay in `v1`.
- **The service layer is not versioned.** `v1` and `v2` handlers both call the same
  `services/<domain>` interface. Each version owns its own `models.go` and does its own
  conversion to/from the entity. If two versions genuinely need different behavior, branch inside
  the service by an explicit argument — don't fork the service.
- Delete a version only after its consumers are gone. Keeping `v1` and `v2` side by side is the
  point of the layout.

## Testing

The interface-per-layer rule exists so each layer is testable in isolation:

- **Services** are tested against **mock datastores and mock peer-domain services** — implement
  the interface with a fake, inject it, assert on business behavior. No real DB needed.
- **Handlers** are tested with a mock service and `net/http/httptest`, asserting the
  request⇄entity⇄response conversion and status codes.
- **Datastore implementations** get integration tests against a real (or containerized) DB, since
  their whole job is the DB round-trip and the row⇄entity mapping.
- Keep mocks next to the interface they mock (e.g. a `mock_orders_test.go` in the service package,
  or generate them with `mockgen`). Table-driven tests are the Go norm here.
