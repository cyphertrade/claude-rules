---
paths:
  - "src/grpc/grpc_server/**/*"
  - "build.rs"
  - "proto/**/*"
---

# gRPC Server Integration (tonic + prost + ci-utils + yft-service-sdk)

## Constraints

- MUST enable both `grpc` and `macros` features on `yft-service-sdk`
- MUST include `tonic-prost = "*"` alongside `tonic`, `prost`, `prost-types`
- MUST add `ci-utils` under `[build-dependencies]`, not `[dependencies]`
- MUST implement every `rpc` method from the proto as an async method in the trait impl
- MUST return `Ok(tonic::Response::new(...))` for all responses — business errors go inside `oneof`, not `tonic::Status`
- MUST implement error mapping via `From` traits, not inline in gRPC methods
- MUST call `configure_grpc_server` before `start_application().await`
- MUST register all gRPC services inside the same `add_grpc_services` closure
- NEVER put business logic in gRPC server methods — they are adapters between network models and flows
- NEVER start a separate tonic server manually — let `ServiceContext` own the lifecycle
- NEVER leak `tonic::Request`/`Response` types into domain logic
- Reserve `Err(tonic::Status)` only for transport/infrastructure failures where no valid response can be produced
- Prefer matching `tonic`/`prost` versions across services
- If unsure which proto repo/URL or filename to use, ASK instead of guessing

## Setup

### Cargo.toml

```toml
[build-dependencies]
ci-utils = { git = "https://github.com/ITYFT/ci-utils.git", tag = "0.1.4" }

[dependencies]
yft-service-sdk = { path = "yft-service-sdk", features = ["grpc", "macros" /* other features */] }

# gRPC core (required stack)
tonic-prost = "*"

tonic = "X.Y"
prost = "X.Y"
prost-types = "X.Y"
```

### build.rs

Place `build.rs` in the crate root next to `Cargo.toml`. Proto files are downloaded and compiled at build time via `ci-utils` — never store them manually.

```rust
fn main() {
    let url = "https://raw.githubusercontent.com/ITYFT/trading-engine-proto-contracts/master/";

    ci_utils::sync_and_build_proto_file_with_builder(url, "TradingEngineAccounts.proto", |x| {
        x.type_attribute(".", "#[derive(serde::Serialize,serde::Deserialize)]")
    });
}
```

`ci-utils` downloads protos into `proto/` at the crate root and invokes `prost_build`. The closure adds type-level attributes (e.g. serde derives) to all generated types.

### Including Generated Code

```rust
pub mod trading_engine_grpc {
    tonic::include_proto!("trading_engine");
}
```

- Module name: `<proto_package>_grpc` (e.g. `trading_engine_grpc`)
- Argument to `include_proto!`: the exact `package` name from the `.proto` file

## Server Struct

Place server implementations in `src/grpc/grpc_server/<service_name>.rs`.

Name pattern: `<ServiceName>GrpcServer`, wrapping `Arc<AppContext>`.

```rust
use std::sync::Arc;
use crate::app::AppContext;

pub struct AccountsGrpcServer(pub Arc<AppContext>);
```

## Implementing the Service Trait

Each gRPC server method follows the adapter pattern: unwrap request, map to DTO, call flow via `self.0` (AppContext), map result to gRPC response.

```rust
use crate::trading_engine_grpc::trading_engine_accounts_service_server::TradingEngineAccountsService;
use crate::trading_engine_grpc::{CreateAccountGrpcRequest, CreateAccountGrpcResponse};
use crate::app::AppContext;
use std::sync::Arc;

pub struct AccountsGrpcServer(pub Arc<AppContext>);

#[tonic::async_trait]
impl TradingEngineAccountsService for AccountsGrpcServer {
    async fn create_account(
        &self,
        request: tonic::Request<CreateAccountGrpcRequest>,
    ) -> Result<tonic::Response<CreateAccountGrpcResponse>, tonic::Status> {
        let request = request.into_inner();

        // delegate to flow, using AppContext
        let create_result = create_account(self.0.as_ref(), request.clone()).await;

        // map flow result into gRPC response type
        let response = CreateAccountGrpcResponse::from(create_result);

        Ok(tonic::Response::new(response))
    }
}
```

Method signature: `&self` + `tonic::Request<...>` -> `Result<tonic::Response<...>, tonic::Status>`.

## Error Handling

### Business Errors via oneof

If the response proto has a `oneof` with an `Error` variant, always return `Ok(Response)` with the error inside the oneof:

Proto:

```proto
rpc VerifyUserCredentials (VerifyUserCredentialsGrpcRequest) returns (VerifyUserCredentialsGrpcResponse);

message VerifyUserCredentialsGrpcRequest{
    string Email = 1;
    string Password = 2;
}

message VerifyUserCredentialsGrpcResponse{
    oneof VerifyUserCredentialsResult{
        UserGrpcModel User = 1;
        UsersGrpcError Error = 2;
    }
}
```

Implementation:

```rust
#[tonic::async_trait]
impl AuthService for AuthGrpcServer {
    async fn verify_user_credentials(
        &self,
        request: tonic::Request<VerifyUserCredentialsGrpcRequest>,
    ) -> Result<tonic::Response<VerifyUserCredentialsGrpcResponse>, tonic::Status> {
        let req = request.into_inner();

        let domain_result = verify_user_credentials_flow(
            self.0.as_ref(),
            req.email,
            req.password,
        ).await;

        let resp = match domain_result {
            Ok(user) => VerifyUserCredentialsGrpcResponse {
                verify_user_credentials_result: Some(
                    verify_user_credentials_grpc_response::VerifyUserCredentialsResult::User(
                        UserGrpcModel::from(user),
                    ),
                ),
            },
            Err(domain_err) => VerifyUserCredentialsGrpcResponse {
                verify_user_credentials_result: Some(
                    verify_user_credentials_grpc_response::VerifyUserCredentialsResult::Error(
                        domain_err.into(),
                    ),
                ),
            },
        };

        Ok(tonic::Response::new(resp))
    }
}
```

### Error Mapping via From

Centralize error conversions with `From` impls — use `.into()` in handlers:

```rust
// Domain layer
pub enum DomainError {
    InvalidCredentials,
    UserBlocked,
}

// Protobuf-generated (or wrapper) gRPC error type
pub struct UsersGrpcError {
    pub code: i32,
    pub message: String,
}

impl From<DomainError> for UsersGrpcError {
    fn from(err: DomainError) -> Self {
        match err {
            DomainError::InvalidCredentials => Self {
                code: 1,
                message: "Invalid email or password".to_string(),
            },
            DomainError::UserBlocked => Self {
                code: 2,
                message: "User is blocked".to_string(),
            },
        }
    }
}
```

Usage in a gRPC method:

```rust
let resp = match domain_result {
    Ok(user) => /* ... */,
    Err(domain_err) => VerifyUserCredentialsGrpcResponse {
        verify_user_credentials_result: Some(Result::Error(domain_err.into())),
    },
};

Ok(tonic::Response::new(resp))
```

## Server-Streaming RPCs

For streaming responses, use `generate_server_stream!` before the trait impl:

```proto
rpc GetClientAccounts(GetTraderAccountsGrpcRequest) returns (stream TradingEngineAccountGrpcModel);
```

```rust
generate_server_stream!(stream_name: "GetClientAccountsStream", item_name: "TradingEngineAccountGrpcModel");
```

- `stream_name`: `<RpcMethodName>Stream` (e.g. `GetClientAccounts` -> `GetClientAccountsStream`)
- `item_name`: exact gRPC response message type from the `stream`

Same adapter pattern applies: request -> DTO -> flow -> map DTOs to gRPC models in the stream.

## Wiring (main.rs)

Bind servers via `ServiceContext::configure_grpc_server`, then call `start_application()`:

```rust
use std::sync::Arc;
use yft_service_sdk::{GrpcServerBuilder, ServiceContext};
use crate::trading_engine_grpc::trading_engine_accounts_service_server::TradingEngineAccountsServiceServer;
use crate::grpc::grpc_server::accounts_grpc_server::AccountsGrpcServer;

#[tokio::main]
async fn main() {
    let settings_reader = SettingsReader::new(".my-cfd-platform").await;
    let settings_reader = Arc::new(settings_reader);

    let mut service_context = ServiceContext::new(settings_reader.clone()).await;
    let app_context = Arc::new(AppContext::new(&service_context, settings_reader.clone()).await);

    // Configure and bind gRPC services
    service_context.configure_grpc_server(|builder: &mut GrpcServerBuilder| {
        builder.add_grpc_services(|x| {
            x.add_service(TradingEngineAccountsServiceServer::new(AccountsGrpcServer(
                app_context.clone(),
            )))
        });
    });

    service_context.start_application().await;
}
```

- `TradingEngineAccountsServiceServer` — generated tonic server type: `<ProtoServiceName>Server`
- `AccountsGrpcServer` — your struct implementing the gRPC trait, holding `Arc<AppContext>`
- If multiple gRPC services are needed, register all inside the same `add_grpc_services` closure
