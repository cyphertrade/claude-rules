---
paths:
  - "src/**/*nosql*"
  - "src/**/*writer*"
  - "src/adapters/**/*"
---

# MyNoSql Writer (REST API)

## Constraints

- **MUST** use the writer (not reader) to access data **before service start** — the reader requires sync after startup
- **MUST** store writers in `AppContext` (or repositories on `AppContext`) — NEVER construct in controllers
- **MUST** use shared entity types from model crates — NEVER define MyNoSql entities inside services
- **MUST** map DTO <-> entity — NEVER expose raw MyNoSql entities to network models
- Controllers **MUST NOT** call writer methods directly — delegate to flows/repositories
- If table logic becomes non-trivial → wrap writer in a repository struct on `AppContext`

## When to Use Writer vs Reader

- **Writer (REST)** — create/update/delete data, admin operations, reads **before** service startup, direct centralized state
- **Reader (TCP)** — fast in-memory reads **after** the service is running and synced

## Setup

Enable the feature in `Cargo.toml`:

```toml
[dependencies]
yft-service-sdk = { path = "yft-service-sdk", features = ["my-nosql-data-writer-sdk"] }
```

Add the writer endpoint to `src/settings.rs`:

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
    pub my_no_sql_tcp_reader: String,
    pub my_no_sql_writer: String,
}
```

## Creating a Writer

Entity types must implement `MyNoSqlEntity + MyNoSqlEntitySerializer + Sync + Send + 'static`.

```rust
use std::sync::Arc;
use yft_service_sdk::external::my_no_sql_sdk::{
    abstractions::{DataSynchronizationPeriod, MyNoSqlEntity, MyNoSqlEntitySerializer},
    writer::{CreateTableParams, MyNoSqlDataWriter},
};
use crate::settings::SettingsReader;

pub fn create_groups_writer(
    settings: Arc<SettingsReader>,
) -> MyNoSqlDataWriter<TradingGroupNoSqlEntity> {
    MyNoSqlDataWriter::new(
        settings.clone(),
        Some(CreateTableParams {
            persist: true,
            max_partitions_amount: None,
            max_rows_per_partition_amount: None,
        }),
        DataSynchronizationPeriod::Asap,
    )
}
```

## Startup / Before-Service-Start Pattern

Use the writer to read or prepare data before `start_application()` — the reader's in-memory cache is not available yet.

```rust
#[tokio::main]
async fn main() {
    let settings_reader = Arc::new(SettingsReader::new(".my-cfd-platform").await);

    // Create writer before service start for startup checks / migrations
    let groups_writer = MyNoSqlDataWriter::<TradingGroupNoSqlEntity>::new(
        settings_reader.clone(),
        Some(CreateTableParams {
            persist: true,
            max_partitions_amount: None,
            max_rows_per_partition_amount: None,
        }),
        DataSynchronizationPeriod::Asap,
    );

    // Ensure table exists before service starts
    groups_writer
        .create_table_if_not_exists(&CreateTableParams {
            persist: true,
            max_partitions_amount: None,
            max_rows_per_partition_amount: None,
        })
        .await
        .expect("failed to ensure groups table");

    // Then continue with usual wiring
    let mut service_context = ServiceContext::new(settings_reader.clone()).await;
    let app_context = Arc::new(AppContext::new(&service_context, settings_reader.clone()).await);

    service_context.start_application().await;
}
```

## Wiring into AppContext

Add writer fields:

```rust
use my_no_sql_sdk::writer::MyNoSqlDataWriter;

#[derive(Clone)]
pub struct AppContext {
    pub groups_writer: Arc<MyNoSqlDataWriter<TradingGroupNoSqlEntity>>,
    // ... other fields
}
```

Initialize in `AppContext::new`:

```rust
impl AppContext {
    pub async fn new(
        sc: &ServiceContext,
        settings_reader: Arc<SettingsReader>,
    ) -> Self {
        let groups_writer = Arc::new(MyNoSqlDataWriter::<TradingGroupNoSqlEntity>::new(
            settings_reader.clone(),
            Some(CreateTableParams {
                persist: true,
                max_partitions_amount: None,
                max_rows_per_partition_amount: None,
            }),
            DataSynchronizationPeriod::Asap,
        ));

        Self {
            groups_writer,
            // ... other dependencies
        }
    }
}
```

Use via `AppContext` in flows:

```rust
pub async fn update_group_flow(
    app: &AppContext,
    input: UpdateGroupDto,
) -> Result<(), FlowError> {
    let entity = TradingGroupNoSqlEntity::from(input);

    app.groups_writer
        .insert_or_replace_entity(&entity)
        .await
        .map_err(FlowError::from)?;

    Ok(())
}
```

## Layer Rules

- **Controllers** — MUST NOT call writer directly. Map network models <-> DTOs, call flows.
- **Flows** — MAY use writer via `AppContext` for insert/update/delete/bulk/reads (especially before reader warmup). No URLs or HTTP details.
- **Repositories** — Wrap writer for table-specific logic (e.g., `save_group`, `delete_group`, `load_all_groups`). Flows call repo methods.

## Writer API

```rust
impl<TEntity: MyNoSqlEntity + MyNoSqlEntitySerializer + Sync + Send> MyNoSqlDataWriter<TEntity> {
    pub async fn create_table(&self, params: CreateTableParams) -> Result<(), DataWriterError>;
    pub async fn create_table_if_not_exists(&self, params: &CreateTableParams) -> Result<(), DataWriterError>;

    pub async fn insert_entity(&self, entity: &TEntity) -> Result<(), DataWriterError>;
    pub async fn insert_or_replace_entity(&self, entity: &TEntity) -> Result<(), DataWriterError>;
    pub async fn bulk_insert_or_replace(&self, entities: &[TEntity]) -> Result<(), DataWriterError>;

    pub async fn get_entity(&self, partition_key: &str, row_key: &str, update_read_statistics: Option<UpdateReadStatistics>) -> Result<Option<TEntity>, DataWriterError>;
    pub async fn get_by_partition_key(&self, partition_key: &str, update_read_statistics: Option<UpdateReadStatistics>) -> Result<Option<Vec<TEntity>>, DataWriterError>;
    pub async fn get_by_row_key(&self, row_key: &str) -> Result<Option<Vec<TEntity>>, DataWriterError>;
    pub async fn get_all(&self) -> Result<Option<Vec<TEntity>>, DataWriterError>;

    pub async fn delete_row(&self, partition_key: &str, row_key: &str) -> Result<Option<TEntity>, DataWriterError>;
    pub async fn delete_partitions(&self, partition_keys: &[&str]) -> Result<(), DataWriterError>;

    pub async fn clean_table_and_bulk_insert(&self, entities: &[TEntity]) -> Result<(), DataWriterError>;
    pub async fn clean_partition_and_bulk_insert(&self, partition_key: &str, entities: &[TEntity]) -> Result<(), DataWriterError>;
}
```

## Recommended Patterns

- `insert_or_replace_entity` — idempotent upserts
- `clean_table_and_bulk_insert` / `clean_partition_and_bulk_insert` — atomic reset for small config tables
- `create_table_if_not_exists` — startup/init flows to ensure tables exist
- `get_*` variants — exact centralized state reads, especially at startup
