---
paths:
  - "src/http/**/*"
  - "src/graphql/**/*"
---

# HTTP & GraphQL (Axum + async-graphql)

## Constraints

- NEVER put business logic in HTTP handlers or GraphQL resolvers — delegate to flows
- MUST receive `State<Arc<AppContext>>` in all Axum handlers
- MUST register all HTTP routes on a single `Router` passed to `ServiceContext::init_http_router`
- MUST attach `.with_state(app_context.clone())` to the router
- MUST confirm GraphQL endpoint paths with the developer before adding
- Keep HTTP and GraphQL as thin transport layers — no protocol-specific types in domain logic

## Architecture Layers

- **HTTP handlers / GraphQL resolvers** — map network models to DTOs, call flows, map results back
- **Flows** — business logic, agnostic of HTTP/GraphQL
- **Repositories** — data access via `AppContext`, no knowledge of transport layer

---

## HTTP (Axum)

### Folder Layout

```text
src/http/<domain>/
  http_models.rs        # request/response structs (JSON bodies, query params)
  create_user.rs        # one file per endpoint
  login_user.rs
  get_user.rs
```

### Handler Pattern

Handlers accept network models + state, map to DTOs, call flows, map results back:

```rust
use std::sync::Arc;
use axum::extract::State;
use axum::Json;
use crate::app::AppContext;
use crate::http::users::http_models::{CreateUserHttpRequest, CreateUserHttpResponse};
use crate::flows::users::create_user_flow;

pub async fn create_user(
    State(app_context): State<Arc<AppContext>>,
    Json(payload): Json<CreateUserHttpRequest>,
) -> Result<Json<CreateUserHttpResponse>, axum::http::StatusCode> {
    let dto = payload.into_dto(); // network → DTO (if needed)

    let result = create_user_flow(app_context.as_ref(), dto)
        .await
        .map_err(|err| err.into_http_status())?;

    Ok(Json(CreateUserHttpResponse::from(result)))
}
```

### Route Registration (main.rs)

```rust
use std::sync::Arc;
use axum::routing::post;
use yft_service_sdk::external::axum::Router;
use crate::app::AppContext;
use crate::http::trading::{update_active, close_active_position, place_order, update_pending, cancel_order};

#[tokio::main]
async fn main() {
    let settings_reader = SettingsReader::new(".my-cfd-platform").await;
    let settings_reader = Arc::new(settings_reader);

    let mut service_context = yft_service_sdk::ServiceContext::new(settings_reader.clone()).await;
    let app_context = Arc::new(AppContext::new(&service_context, settings_reader.clone()).await);

    service_context.init_http_router(
        Router::new()
            .route("/api/trading/active", post(update_active))
            .route("/api/trading/active/close", post(close_active_position))
            .route("/api/trading/order/place", post(place_order))
            .route("/api/trading/order", post(update_pending))
            .route("/api/trading/order/cancel", post(cancel_order))
            .with_state(app_context.clone()),
    );

    service_context.start_application().await;
}
```

Path naming follows `/api/<domain>/<action>` — confirm exact routes with the developer.

---

## GraphQL (async-graphql)

### Folder Layout

```text
src/graphql/<domain>/
  <domain>_query.rs       # Query root
  <domain>_mutation.rs    # Mutation root
  models/
    <target>_models.rs    # GraphQL objects/enums
  mod.rs                  # schema builder
```

### Building a Domain Schema

Each domain exposes a schema builder in `src/graphql/<domain>/mod.rs`:

```rust
use async_graphql::{Schema, EmptySubscription};
use crate::graphql::reports::reports_query::ReportsQueryRoot;
use crate::graphql::reports::reports_mutation::ReportsMutationRoot;

pub type ReportsSchema = Schema<ReportsQueryRoot, ReportsMutationRoot, EmptySubscription>;

pub fn get_reports_schema() -> ReportsSchema {
    Schema::build(ReportsQueryRoot, ReportsMutationRoot, EmptySubscription)
        // .data(...) if you want to inject global data
        .finish()
}
```

Naming: type alias `<DomainName>Schema`, builder `get_<domain_name>_schema`.

### GraphiQL & Handler

Define top-level GraphQL HTTP endpoints in `src/graphql/mod.rs`:

```rust
use async_graphql::{
    http::GraphiQLSource,
    Request as GraphQLRequest,
    Response as GraphQLResponse,
};
use async_graphql_axum::GraphQLRequest as AxumGraphQLRequest;
use async_graphql_axum::GraphQLResponse as AxumGraphQLResponse;
use axum::{response, extract::State};
use axum::response::IntoResponse;
use http::HeaderMap;

use crate::graphql::reports::ReportsSchema;
use crate::metrics::{HTTP_REQUESTS_COUNTER, GRAPHQL_REQUESTS_DURATION_HISTORGRAM};
use crate::graphql::RequestHost;

pub async fn graphiql() -> impl IntoResponse {
    response::Html(GraphiQLSource::build().endpoint("/api/reports/ql").finish())
}
```

GraphQL handler with metrics:

```rust
pub async fn graphql_handler(
    State(schema): State<ReportsSchema>,
    headers: HeaderMap,
    req: AxumGraphQLRequest,
) -> AxumGraphQLResponse {
    let req: GraphQLRequest = req.into_inner();
    let host = headers
        .get("host")
        .and_then(|h| h.to_str().ok())
        .unwrap_or_default()
        .to_string();

    let operation_name = req.operation_name.clone().unwrap_or_else(|| "N/A".to_string());

    HTTP_REQUESTS_COUNTER
        .with_label_values(&[&operation_name])
        .inc();

    let start = GRAPHQL_REQUESTS_DURATION_HISTORGRAM
        .with_label_values(&[&operation_name])
        .start_timer();

    let response: GraphQLResponse = schema
        .execute(req.data(RequestHost(host)))
        .await;

    start.observe_duration();
    response.into()
}
```

- If metrics (`HTTP_REQUESTS_COUNTER`, `GRAPHQL_REQUESTS_DURATION_HISTORGRAM`) exist in the project, use them
- If not present → add following existing metrics patterns, or omit per team decision
- Pass `RequestHost(host)` via `.data(...)` if resolvers depend on host info

### GraphQL Route Registration (main.rs)

```rust
use std::sync::Arc;
use axum::routing::get;
use yft_service_sdk::external::axum::Router;
use crate::graphql::{self, get_reports_schema};
use crate::app::AppContext;

#[tokio::main]
async fn main() {
    let settings_reader = SettingsReader::new(".my-cfd-platform").await;
    let settings_reader = Arc::new(settings_reader);

    let mut service_context = yft_service_sdk::ServiceContext::new(settings_reader.clone()).await;
    let app_context = Arc::new(AppContext::new(&service_context, settings_reader.clone()).await);

    let reports_schema = get_reports_schema();

    service_context.init_http_router(
        Router::new()
            .route(
                "/api/reports/ql",
                get(graphql::graphiql).post(graphql::graphql_handler),
            )
            .with_state(reports_schema)
            .with_state(app_context.clone()),
    );

    service_context.start_application().await;
}
```

- GraphQL endpoint path MUST match the one used in `graphiql()`
- Provide `ReportsSchema` as Axum state via `.with_state(reports_schema)`
- If resolvers get everything through schema data, `AppContext` as separate state may be unnecessary
