---
paths:
  - "src/grpc/grpc_client/**/*"
  - "build.rs"
  - "proto/**/*"
---

# gRPC Client Integration (tonic + yft-service-sdk macros)

## Constraints

- MUST enable both `grpc` and `macros` features on `yft-service-sdk`
- MUST include `tonic-prost = "*"` alongside `tonic`, `prost`, `prost-types`
- MUST have a corresponding `tonic::include_proto!` module matching the `crate_ns` in the macro
- MUST use the same proto contract and package for server and client of the same service
- NEVER manually implement low-level client logic — always use `generate_grpc_client` macro
- NEVER construct gRPC clients directly inside controllers/handlers — obtain via `AppContext`
- NEVER propagate `tonic::Status` into the domain — map to your own error types
- NEVER call gRPC clients directly from controllers — use flows or repository wrappers
- Keep gRPC as a transport detail: domain logic and DTOs stay separate from tonic types
- If unsure which proto repo/URL to use, ASK instead of guessing

## Setup

### Cargo.toml

```toml
[dependencies]
yft-service-sdk = { path = "yft-service-sdk", features = ["grpc", "macros" /* other features */] }

# If we work with gRPC, tonic-prost = "*" is required
tonic = "*"
prost = "*"
prost-types = "*"
tonic-prost = "*"
```

### build.rs

Reuse the same proto-generation pipeline as gRPC servers. If already configured for a server, no changes needed.

```rust
fn main() {
    let url = "https://raw.githubusercontent.com/ITYFT/trading-engine-proto-contracts/master/";

    ci_utils::sync_and_build_proto_file_with_builder(url, "TradingEnginePersistence.proto", |x| {
        x.type_attribute(".", "#[derive(serde::Serialize,serde::Deserialize)]")
    });
}
```

## Client Struct

Define client structs in `src/grpc/grpc_client/<client_name>.rs`.

Name pattern: `<ServiceName>GrpcClient` (e.g. `TradingEnginePersistenceGrpcClient`).

```rust
yft_service_sdk::macros::use_grpc_client!();

#[yft_service_sdk::external::my_grpc_extensions::client::generate_grpc_client(
    proto_file: "./trading-engine/proto/TradingEnginePersistence.proto",
    crate_ns: "crate::trading_engine_persistence_grpc",
    retries: 3,
    request_timeout_sec: 3,
    ping_timeout_sec: 1,
    ping_interval_sec: 3,
)]
pub struct TradingEnginePersistenceGrpcClient;
```

Macro parameters:
- `proto_file` — path to compiled proto relative to crate root
- `crate_ns` — Rust module path where `tonic::include_proto!` lives
- `retries` — automatic retry attempts
- `request_timeout_sec` — per-request timeout
- `ping_timeout_sec` — health/ping check timeout
- `ping_interval_sec` — interval between pings

## Settings

Add a `*_grpc` URL field to `SettingsModel` for each client:

```rust
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
    pub my_sb_tcp_host_port: String,
    pub my_no_sql_writer: String,
    pub my_no_sql_tcp_reader: String,

    // gRPC endpoint for Trading Engine Persistence service
    pub engine_persistence_grpc: String,
}
```

### GrpcClientSettings Implementation

Implement `GrpcClientSettings` on `SettingsReader` — match `name` against `<ClientStruct>::get_service_name()`. For multiple clients, extend the `if`/`match` chain.

```rust
use yft_service_sdk::external::my_grpc_extensions::client::GrpcClientSettings;
use crate::grpc::grpc_client::trading_engine_persistence::TradingEnginePersistenceGrpcClient;

pub struct GrpcUrl(pub String);

#[async_trait::async_trait]
impl GrpcClientSettings for SettingsReader {
    async fn get_grpc_url(&self, name: &'static str) -> String {
        let settings = self.get_settings().await;

        if TradingEnginePersistenceGrpcClient::get_service_name() == name {
            return settings.engine_persistence_grpc;
        }

        panic!("Settings not found");
    }
}
```

The `generate_grpc_client` macro provides `get_service_name()` — never hardcode the service name string.

## Wiring (AppContext)

Store clients in `AppContext` wrapped in `Arc`. Flows and repositories obtain clients from `AppContext`.

```rust
use std::sync::Arc;
use crate::grpc::grpc_client::trading_engine_persistence::TradingEnginePersistenceGrpcClient;

#[derive(Clone)]
pub struct AppContext {
    pub engine_persistence_client: Arc<TradingEnginePersistenceGrpcClient>,
    // ... other fields
}

impl AppContext {
    pub async fn new(
        sc: &ServiceContext,
        settings_reader: Arc<SettingsReader>,
    ) -> Self {
        let engine_persistence_client =
            Arc::new(TradingEnginePersistenceGrpcClient::new(settings_reader.clone()));

        Self {
            engine_persistence_client,
            // ... init other dependencies
        }
    }
}
```

## Layering

- **Controllers** — MUST NOT call gRPC clients directly. Work with network models / DTOs and call flows.
- **Flows** — MAY use gRPC clients via `AppContext`. Work with DTOs; no `tonic::Request`/`Response` types in domain.
- **Repositories / adapters** — If integration is complex, wrap the gRPC client (e.g. `EnginePersistenceRepository` exposing domain methods internally using the client).

### Flow Usage Example

```rust
pub async fn sync_account_to_persistence(
    app: &AppContext,
    input: SyncAccountDto,
) -> Result<(), FlowError> {
    let mut client = app.engine_persistence_client.clone();

    // Build gRPC request model from DTO
    let request_model = input.into_grpc_model();

    client
        .sync_account(request_model)
        .await
        .map_err(FlowError::from)?;

    Ok(())
}
```
