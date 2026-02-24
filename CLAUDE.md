# YFT Rust Microservices

Rust microservices monorepo using `yft-service-sdk` for shared infrastructure.

## Hard constraints

- NEVER output code that would fail `cargo build` — mentally compile first
- NEVER create files or structures beyond what was explicitly requested — ask instead of guessing
- NEVER use mapper functions — use `From`/`TryFrom` trait conversions only
- NEVER validate inside DTOs or flows — `validator` crate on network models only
- NEVER bypass `ServiceContext` or `AppContext` when adding dependencies
- MUST use `anyhow` for error context at library boundaries
- MUST resolve `{placeholder}` to concrete domain names — never use braces literally

## Architecture

```
Transport (src/api/, src/http/, src/grpc/) — network models, serde + validator
    ↕ From/TryFrom
Domain (src/flows/{domain}/, src/dto/{domain}/) — DTOs, no serde
    ↕ From/TryFrom
Data access (src/adapters/, src/postgres/) — data store models
```

Dependencies wired through `AppContext` (`src/app/app_ctx.rs`), runtime via `ServiceContext`.

## Feature rules (auto-loaded by file path)

| Integration | Rule file | Triggers on |
|---|---|---|
| Core architecture | `basic-instructions.md` | `src/**/*`, `Cargo.toml` |
| gRPC client | `grpc-client.md` | `src/grpc/grpc_client/**/*`, `build.rs`, `proto/**/*` |
| gRPC server | `grpc-server.md` | `src/grpc/grpc_server/**/*`, `build.rs`, `proto/**/*` |
| HTTP / GraphQL | `http-graphql.md` | `src/http/**/*`, `src/graphql/**/*` |
| MyNoSql reader | `nosql-reader.md` | `src/**/*nosql*`, `src/**/*reader*`, `src/adapters/**/*` |
| MyNoSql writer | `nosql-writer.md` | `src/**/*nosql*`, `src/**/*writer*`, `src/adapters/**/*` |
| PostgreSQL | `postgresql-sqlx.md` | `src/postgres/**/*`, `migrations/**/*` |
| ServiceBus | `service-bus.md` | `src/background/**/*`, `src/**/*publisher*`, `src/**/*subscriber*` |

When creating **new** integrations (files don't exist yet), read the relevant rule file first.
