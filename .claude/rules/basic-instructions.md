---
paths:
  - "src/**/*"
  - "Cargo.toml"
---

# YFT Rust Service Architecture & service-sdk Rules

## Hard constraints

- NEVER output Rust code that would fail `cargo build`. Mentally compile before outputting — verify imports, modules, paths, types, trait bounds, derives, and syntax. Fix issues or ask for clarification first.
- NEVER create files, modules, or structures beyond what was explicitly requested. If asked to create a DTO, create only the DTO. If something else seems needed, ask.
- NEVER create placeholder implementations, mocks, stubs, or "helper" modules unless explicitly requested.
- NEVER use `{placeholder}` literally as a folder/file name — always resolve to a concrete domain name (e.g. `users`, `trades`, `affiliates`).
- NEVER bypass `ServiceContext` or `AppContext` when adding dependencies.
- NEVER put business logic in `main.rs`, controllers, or adapters.
- NEVER use free mapper functions (`to_dto()`, `from_dto()`, `map_*()`) — use trait conversions only (see Mapping rules).
- NEVER derive `serde::Serialize`/`serde::Deserialize` on DTOs unless explicitly required for internal persistence.
- NEVER validate inside DTOs or flows — validation belongs in the transport layer only.
- NEVER use `anyhow` — use typed service errors (`src/error.rs`). If existing code uses `anyhow`, refactor it to typed errors.
- NEVER start custom HTTP/gRPC servers outside `ServiceContext`.
- NEVER duplicate health/metrics endpoints — use SDK-provided ones.
- NEVER create custom tracing/logging setups that conflict with `ServiceContext`.
- MUST wrap external library errors into typed service errors via `From` impls — never use `anyhow`.
- MUST place service errors in `src/error.rs`.
- MUST use `validator` crate for all input validation (network models only).
- MUST follow existing repo patterns over inventing new ones.
- MUST keep changes small and focused to the relevant module/layer.
- When missing critical info (entity name, table, route path), ASK a clarifying question instead of guessing.

## Architecture

### Layered model

```
Transport (src/api/, src/transport/)
    ↕  network models (request/response) — serde + validator derives
    ↕  From/TryFrom conversions
Domain (src/flows/{domain}/, src/dto/{domain}/)
    ↕  DTOs — no serde, no validation
    ↕  From/TryFrom conversions
Data access (src/adapters/)
    ↕  data store models — DB/NoSQL representations
```

### Three model types

| Model | Lives in | Purpose | Rules |
|-------|----------|---------|-------|
| Network models | `src/api/` or `src/transport/` | HTTP/gRPC request/response payloads | Derive `serde` + `validator::Validate`. Never used in flows or repos. |
| DTOs | `src/dto/{domain}/` | Internal domain types for flows | No serde, no validation. Transport- and storage-agnostic. One DTO per file (unless extremely coupled). |
| Data store models | `src/adapters/db/`, `src/adapters/nosql/` | DB/NoSQL row representations | Used by repos only. Mapped to/from DTOs via traits. Never exposed to controllers. |

### Request lifecycle

1. Deserialize into network request model (transport layer)
2. `req.validate()?` immediately
3. Convert request → DTO via `From`/`TryFrom`
4. Call flow with `&AppContext` + DTO
5. Convert result DTO → network response via `From`/`TryFrom`
6. Return response

## Service wiring

Every service uses `yft-service-sdk` with `ServiceContext` for runtime (HTTP/gRPC servers, logging, metrics, health endpoints) and `AppContext` as the dependency container.

### Required Cargo dependencies

```toml
[dependencies]
tokio = { version = "X.Y", features = ["macros", "rt-multi-thread"] }
serde = { version = "X.Y", features = ["derive"] }
```

### Settings (`src/settings.rs`)

```rust
use yft_service_sdk::external::my_settings_reader;

#[derive(
    Debug,
    Clone,
    serde::Serialize,
    serde::Deserialize,
    yft_service_sdk::macros::SdkSettingsTraits,
    yft_service_sdk::macros::AutoGenerateSettingsTraits,
    yft_service_sdk::external::my_settings_reader::SettingsModel,
)]
pub struct SettingsModel {
    pub logs_host_port: String,
}
```

- Extend `SettingsModel` with new fields as needed — do not create separate settings types.
- Keep `SettingsModel` purely declarative (no logic).
- Always read config through the generated `SettingsReader`, never from env vars directly.

### AppContext (`src/app/app_ctx.rs`)

Central dependency container holding repos, caches, clients, bus clients, config values, and shared state.

```rust
pub async fn new(
    sc: &ServiceContext,
    settings_reader: Arc<SettingsReader>,
) -> Self {
    // build and return AppContext
}
```

- Add new dependencies as fields on `AppContext`, initialized in `AppContext::new(...)`.
- Do not create alternative context types, construct deps inside handlers, or put transport logic here.

### main.rs template

```rust
use std::sync::Arc;
use yft_service_sdk::ServiceContext;

#[tokio::main]
async fn main() {
    // 1. Load settings
    let settings_reader = SettingsReader::new(".my-cfd-platform").await;
    let settings_reader = Arc::new(settings_reader);

    // 2. Initialize the shared service runtime
    let mut service_context = ServiceContext::new(settings_reader.clone()).await;

    // 3. Build the application context with all dependencies
    let app_context = Arc::new(AppContext::new(&service_context, settings_reader.clone()).await);

    // 4. (Optional) Configure HTTP or gRPC here using service_context and app_context

    // 5. Start the service
    service_context.start_application().await;
}
```

`main.rs` does only: load settings → create `ServiceContext` → build `AppContext` → wire HTTP/gRPC → start. Nothing else.

## Patterns

### Mapping between layers (trait conversions only)

Priority order:
1. `From<Source> for Target` (preferred)
2. `.into()` at call sites
3. `TryFrom` / `.try_into()` for fallible conversions — map errors to typed service errors via `From` impl
4. `Into<ExternalType>` for your local type only when orphan rules prevent `From`

Applies to all boundaries: network ↔ DTO, DTO ↔ data store, DTO ↔ external client models.

### Controllers (thin transport adapters)

```
network request → validate → convert to DTO → call flow → convert result DTO → network response
```

- MUST NOT access databases, caches, or clients directly.
- MUST NOT use data store models.
- MUST NOT contain business logic beyond simple validation and mapping.

### Flows (business logic)

```rust
pub async fn some_flow(
    app: &AppContext,
    input: SomeInputDto,
) -> Result<SomeOutputDto, FlowError> {
    // business logic working only with DTOs
}
```

- Receive DTO input, return DTO output.
- No request/response processing, no DB/HTTP/gRPC code.

### Validation (`validator` crate only)

Network request models:
```rust
#[derive(serde::Deserialize, validator::Validate)]
pub struct SomeRequest {
    #[validate(length(min = 1))]
    pub name: String,
    #[validate(email)]
    pub email: String,
}
```

- Call `req.validate()?` immediately after deserialization in the controller.
- DTOs MUST NOT derive `Validate` or contain validation attributes.
- Flows may check logical constraints (e.g. "user not found") but not format validation.
- If `validator` cannot express a rule, ask before inventing custom validation.

### Error handling (typed errors only — no `anyhow`)

- Service errors live in `src/error.rs` as enums implementing `std::error::Error`.
- Wrap external library errors via `From` impls on the service error enum.
- If existing code uses `anyhow`, refactor it to typed service errors.
- Do not use ad-hoc string mapping or custom helpers.

## Module placement decision tree

| What you're adding | Place it in |
|---|---|
| Flow (business logic) | `src/flows/{domain}/flow_name.rs` |
| DTO (internal domain type) | `src/dto/{domain}/type_name.rs` + `mapper.rs` for trait impls |
| Network request/response model | `src/api/...` or `src/transport/...` |
| DB/NoSQL adapter or repository | `src/adapters/db/...` or `src/adapters/nosql/...` |
| External HTTP client | `src/adapters/http_clients/...` |
| Service dependency wiring | `src/app/app_ctx.rs` + `main.rs` |
| Settings field | `src/settings.rs` (`SettingsModel` struct) |
| Service error types | `src/error.rs` |

If the correct `{domain}` folder is unclear: prefer reusing an existing folder that fits. If none fits, choose a short, clear business-domain name. If still unclear, ask.

## Related rule files

- MyNoSql reader: `.claude/rules/nosql-reader.md`
- MyNoSql writer: `.claude/rules/nosql-writer.md`
- gRPC client: `.claude/rules/grpc-client.md`
- gRPC server: `.claude/rules/grpc-server.md`
- HTTP/GraphQL: `.claude/rules/http-graphql.md`
- PostgreSQL: `.claude/rules/postgresql-sqlx.md`
- ServiceBus: `.claude/rules/service-bus.md`
