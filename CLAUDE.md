# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the Encore monorepo — a multi-language infrastructure SDK and CLI for TypeScript and Go. It contains:
- The `encore` CLI and background daemon (Go)
- The application parser, compiler, and code generator (Go)
- The TypeScript parser (Rust, using SWC)
- The shared runtime core (Rust)
- The Go SDK (`encore.dev` package, sub-module at `runtimes/go/`)
- The JavaScript/TypeScript SDK (`encore.dev` npm package, at `runtimes/js/encore.dev/`)
- The Node.js native runtime bindings (Rust NAPI, at `runtimes/js/src/`)
- End-to-end tests (`e2e-tests/`)

## Build Commands

### Go (main module `encr.dev`)
```bash
go build ./...                        # build all Go packages
go build ./cli/cmd/encore             # build the encore CLI binary
go install ./cli/cmd/git-remote-encore  # install the git remote helper
```

### Rust workspace (tsparser, runtimes/core, runtimes/js, supervisor, miniredis)
```bash
cargo build                           # build all Rust crates
cargo install --path tsparser --force --debug  # install the tsparser binary
go run ./pkg/encorebuild/cmd/build-local-binary encore-runtime.node  # build JS runtime .node binary
```

### TypeScript SDK (`runtimes/js/encore.dev/`)
```bash
cd runtimes/js/encore.dev && npm ci && npm run build
```

## Running Tests

### Environment variables required for most tests
```bash
export ENCORE_RUNTIMES_PATH=/path/to/this/repo/runtimes
export ENCORE_GOROOT=$(encore daemon env | grep ENCORE_GOROOT | cut -d= -f2)
```

### Go tests (main module)
```bash
go test -short -tags=dev_build ./...                     # all tests, skip slow ones
go test -tags=dev_build ./v2/parser/...                  # single package
go test -run TestFoo -tags=dev_build ./v2/parser/...     # single test
```

### Go runtime sub-module tests
```bash
cd runtimes/go && go test -short -tags=dev_build ./...
```

### E2E tests (requires encore-go, tsparser binary, and both env vars)
```bash
go test -short -tags=e2e ./e2e-tests
```
To regenerate golden files used by `internal/clientgen`:
```bash
cd e2e-tests && go test -golden-update
```

### Rust tests
```bash
cargo nextest run    # preferred; or: cargo test
cargo fmt --all --check
cargo clippy --all-targets --all-features -- -D warnings
```

## Linting / Static Analysis

Run the same checks as CI locally:
```bash
./check.bash            # checks only changed files vs origin/main
./check.bash --all      # checks all files
./check.bash --diff     # show the diff being analyzed
```
This requires `go`, `git`, `sed`, `semgrep`, and auto-installs `reviewdog`, `staticcheck`, `errcheck`, `ineffassign` via `go install`.

Go fmt check:
```bash
gofmt -s -d .
```

Non-root Go modules need build tags for tools to work:
```bash
go vet -tags encore,encore_internal,encore_app ./...
```

## Running the Daemon (development)
```bash
./encore daemon -f      # start daemon from locally-built binary
```
Commands like `encore run` must use the **same binary** the daemon is running.

## Repository Architecture

### Top-level layout

| Directory | Language | Purpose |
|-----------|----------|---------|
| `cli/` | Go | `encore` CLI entry points and the background daemon |
| `v2/` | Go | Parser, application model, compiler, and code generator |
| `tsparser/` | Rust | TypeScript/JS static analysis using a forked SWC |
| `runtimes/core/` | Rust | Shared runtime core (HTTP, Pub/Sub, SQL, etc.) |
| `runtimes/js/` | Rust + TS | NAPI bindings for Node.js + the TS SDK package |
| `runtimes/go/` | Go | Go SDK (`encore.dev` package, separate Go module) |
| `supervisor/` | Rust | Process supervisor for running app services |
| `pkg/` | Go | Shared utilities: client gen, builder, dist packaging, etc. |
| `internal/` | Go | Internal utilities (version, config, tracing, etc.) |
| `e2e-tests/` | Go | End-to-end integration tests |
| `proto/` | Protobuf | Wire protocol between CLI and daemon |
| `tools/` | Go | Semgrep rules and release tooling |

### Go application pipeline (`v2/`)

The `v2/` package implements the full build pipeline for Encore apps:

1. **`v2/internals/`** — low-level infrastructure: package loader (`pkginfo`), schema parser, parse context, error types.
2. **`v2/parser/`** — static analysis of Encore applications. Produces a typed resource model for APIs (`parser/apis/`), infra resources (`parser/infra/`), and service structs. The output is an `app.Desc` value.
3. **`v2/app/`** — validation layer: cross-checks the parsed model (APIs, databases, Pub/Sub, caches, etc.) and produces a validated `app.Desc`.
4. **`v2/codegen/`** — generates Go source code (`apigen/`, `infragen/`, `rewrite/`) and CUE config (`cuegen/`). The rewrite pass transforms the user's source to call into the Encore runtime.
5. **`v2/compiler/`** — orchestrates the full Go compile with the rewritten sources.
6. **`v2/tsbuilder/`** — TypeScript-specific build pipeline: invokes `tsparser`, runs `tsbundler-encore`, and wires into the JS runtime.
7. **`v2/v2builder/`** — the combined builder that dispatches to either the Go or TS pipeline.

### CLI and daemon (`cli/`)

- **`cli/cmd/encore/`** — the `encore` CLI binary (subcommands: `run`, `test`, `check`, `build`, `db`, `secret`, etc.).
- **`cli/daemon/`** — the background daemon process responsible for managing app processes, databases (via emulators), and serving the local dashboard. Communicates with CLI via gRPC (protobuf in `proto/`).

### Rust components

- **`tsparser/`** — parses TypeScript/JavaScript files to extract Encore resource declarations (APIs, databases, Pub/Sub, etc.). Uses a fork of SWC (`encoredev/swc`) and a fork of `rust-postgres` (`encoredev/rust-postgres`).
- **`runtimes/core/`** — the Rust runtime implementing HTTP routing, SQL connection pools, Pub/Sub, object storage, caching, secrets, and tracing. Shared between Go apps (via FFI) and Node.js apps (via NAPI).
- **`runtimes/js/src/`** — NAPI bindings that expose `runtimes/core` to Node.js. Compiled to `encore-runtime.node`.
- **`supervisor/`** — lightweight process supervisor that manages service processes for an Encore app.

### Go SDK sub-module (`runtimes/go/`)

A separate Go module (`encore.dev`) containing the public API surface that user apps import: `encore.dev/storage/sqldb`, `encore.dev/pubsub`, `encore.dev/storage/objects`, `encore.dev/storage/cache`, `encore.dev/cron`, `encore.dev/config`, `encore.dev/beta/auth`, `encore.dev/beta/errs`, `encore.dev/middleware`, etc. Tests here use `-tags=dev_build` to hook into the test harness.

### TypeScript SDK (`runtimes/js/encore.dev/`)

The published `encore.dev` npm package. Written in TypeScript, bundled with `tsc`. Tests run with `bun test`. If you change public API, run `npm run docs` and commit the generated docs.

## Multiple Go Modules

The repo has two Go modules:
- `encr.dev` (root) — the CLI, parser, compiler, daemon, and all tooling
- `encore.dev` (at `runtimes/go/`) — the Go SDK used by apps

When running `go` commands from the root, you only affect the root module. Explicitly `cd runtimes/go` to work on the SDK. The `.github/workflows/makefile` runs lint tools in each module separately, which is why paths in reviewdog output include the module-relative prefix.

## Important Conventions

- **Build tags**: Non-root modules and some internal packages require `-tags encore,encore_internal,encore_app` or `-tags dev_build` to compile correctly.
- **Generated folders**: `encore.gen/` and `.encore/` in app directories are CLI-regenerated — never edit them.
- **`encore.app` file**: The app manifest is a CUE-formatted text file (despite the `.app` extension).
- **Protobuf**: Run `go generate ./proto/...` (requires `protoc`) to regenerate Go stubs from `.proto` files.
- **Golden test files**: Test golden files in `e2e-tests/testdata/` and `internal/clientgen/testdata/` are regenerated with `-golden-update`.
