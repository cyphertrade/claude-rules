---
paths:
  - "src/postgres/**/*"
  - "migrations/**/*"
---

# PostgreSQL (sqlx)

## Constraints

- NEVER access `PgPool` or repositories from the network layer (HTTP/gRPC) — go through flows
- NEVER embed SQL queries directly in flow functions — use repositories
- NEVER create `PgPool` manually with `PgPoolOptions` — use `ServiceContext::get_db_pool`
- NEVER edit old migrations already applied to shared environments (prod, UAT) — add new ones
- MUST wrap `PgPool` in `Arc<PgPool>` for sharing between repositories
- MUST create all repositories in `AppContext::new` (or a dedicated factory)
- MUST back every structural DB change with a migration
- MUST keep migrations in sync with models and repositories

## Enabling PostgreSQL

Add to `Cargo.toml`:

```toml
[dependencies]
yft-service-sdk = { path = "yft-service-sdk", features = ["postgresql", /* other features */] }
sqlx = { version = "X.Y", features = ["runtime-tokio-rustls", "postgres"] }
```

Add the connection setting to `src/settings.rs`:

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
    pub my_no_sql_writer: String,
    pub my_no_sql_tcp_reader: String,

    // PostgreSQL connection string (sqlx)
    pub sqlx_connection: String,
}
```

The generated `SettingsReader` implements the `SqlxSettings` trait (from the SDK), exposing connection details for `get_db_pool`.

## Getting a PgPool

`ServiceContext` exposes:

```rust
#[cfg(feature = "postgresql")]
pub async fn get_db_pool<T: crate::SqlxSettings>(
    &self,
    connection_settings: &T,
    config_callback: impl Fn(&mut sqlx::postgres::PgPoolOptions) -> sqlx::postgres::PgPoolOptions,
) -> sqlx::Pool<sqlx::postgres::Postgres> {
    // implementation in yft-service-sdk
}
```

Typical usage:

```rust
use std::sync::Arc;
use sqlx::postgres::PgPool;
use yft_service_sdk::ServiceContext;
use crate::settings::SettingsReader;

pub struct PostgresDeps {
    pub pool: Arc<PgPool>,
}

impl PostgresDeps {
    pub async fn new(settings: &Arc<SettingsReader>, sc: &ServiceContext) -> Self {
        let pool = Arc::new(sc.get_db_pool(settings.as_ref(), |opts| opts.clone()).await);

        Self { pool }
    }
}
```

The `config_callback` allows customizing pool options (max connections, timeouts). When unsure, pass `|opts| opts.clone()`.

## Folder Layout

```text
src/postgres/
  <table_name>/
    <table_name>_model.rs    # data store model(s) representing rows
    <table_name>_repo.rs     # repository with sqlx queries
```

Example:

```text
src/postgres/
  accounts/
    accounts_model.rs
    accounts_repo.rs
  orders/
    orders_model.rs
    orders_repo.rs
```

## Repository Pattern

Each repository holds `Arc<PgPool>` and provides query methods:

```rust
use std::sync::Arc;
use sqlx::postgres::PgPool;

pub struct AccountsRepository {
    pub pool: Arc<PgPool>,
}

impl AccountsRepository {
    pub fn new(pool: Arc<PgPool>) -> Self {
        Self { pool }
    }

    // Example method: customize per table
    // pub async fn get_by_id(&self, id: i64) -> Result<Option<AccountModel>, sqlx::Error> {
    //     sqlx::query_as!(
    //         AccountModel,
    //         r#"
    //             SELECT id, name, created_at
    //             FROM accounts
    //             WHERE id = $1
    //         "#,
    //         id,
    //     )
    //     .fetch_optional(self.pool.as_ref())
    //     .await
    // }
}
```

- Repository methods define whatever queries are needed — no enforced method set
- Keep business logic out — repositories handle data access and model mapping only

## Wiring in AppContext

```rust
use std::sync::Arc;
use sqlx::postgres::PgPool;
use crate::postgres::accounts::accounts_repo::AccountsRepository;

#[derive(Clone)]
pub struct AppContext {
    pub db_pool: Arc<PgPool>,
    pub accounts_repo: Arc<AccountsRepository>,
    // ... other repos and dependencies
}

impl AppContext {
    pub async fn new(
        sc: &ServiceContext,
        settings_reader: Arc<SettingsReader>,
    ) -> Self {
        let db_pool = Arc::new(sc.get_db_pool(settings_reader.as_ref(), |opts| opts.clone()).await);

        let accounts_repo = Arc::new(AccountsRepository::new(db_pool.clone()));

        Self {
            db_pool,
            accounts_repo,
            // ... initialize other repositories
        }
    }
}
```

Handlers/controllers MUST NOT create repositories directly. Flows access data only through repositories on `AppContext`.

---

## Migrations

### Create a Migration

From the service crate root:

```bash
sqlx migrate add table_name
```

Generates a file in `migrations/` with pattern `<YYYYMMDDHHMMSS>_table_name.sql`.

### Write the Migration

Match the SQL to what the code (models/repositories) expects:

```sql
-- 20251122134709_table_name.sql

CREATE TABLE accounts (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

- If adding a column in the model → add a migration for that column
- If dropping/renaming a column → update models, repositories, and migration
- For alterations to existing tables, reflect exactly those changes

### Run Migrations

Run via `sqlx migrate run` or CI/deployment scripts. When adding tables or columns, always include a matching migration so the DB schema matches the Rust models.
