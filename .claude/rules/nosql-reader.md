---
paths:
  - "src/**/*nosql*"
  - "src/**/*reader*"
  - "src/adapters/**/*"
---

# MyNoSql Reader (TCP)

## Constraints

- **MUST** use `MyNoSqlDataReaderTcp` — NEVER use `MyNoSqlDataReader` in services
- **MUST** obtain readers via `sc.get_ns_reader::<T>().await` — NEVER construct `MyNoSqlDataReaderTcp::new(...)` directly
- **MUST** store readers in `AppContext` and pass `AppContext` into flows
- **MUST** import reader via `yft_service_sdk::external::my_no_sql_sdk::reader::MyNoSqlDataReaderTcp`
- Controllers **MUST NOT** access readers directly — only flows and repositories may use them
- Entity types come from shared crates (e.g., `cyphertrade_nosql_contracts`), NEVER defined inside services

## Imports

```rust
use std::sync::Arc;

// Reader import (via service SDK re-export)
use yft_service_sdk::external::my_no_sql_sdk::reader::MyNoSqlDataReaderTcp;

// Entity types from shared models crate
use cyphertrade_nosql_contracts::xp_progression::XpProgressionSettingsMyNoSqlEntity;
use cyphertrade_nosql_contracts::user_group::UserGroupsNoSqlModel;
```

## Setup

Enable the feature in `Cargo.toml`:

```toml
[dependencies]
yft-service-sdk = { path = "yft-service-sdk", features = ["my-nosql-data-reader-sdk"] }
```

Add the TCP reader setting to `src/settings.rs`:

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
}
```

## Entity Requirements

Entity types must implement:

```rust
MyNoSqlEntity + MyNoSqlEntitySerializer + Sync + Send + 'static
```

## Wiring into AppContext

Add reader fields:

```rust
use std::sync::Arc;
use yft_service_sdk::external::my_no_sql_sdk::reader::MyNoSqlDataReaderTcp;

use cyphertrade_nosql_contracts::xp_progression::XpProgressionSettingsMyNoSqlEntity;
use cyphertrade_nosql_contracts::user_group::UserGroupsNoSqlModel;

#[derive(Clone)]
pub struct AppContext {
    pub xp_progression_settings_reader: Arc<MyNoSqlDataReaderTcp<XpProgressionSettingsMyNoSqlEntity>>,
    pub user_groups_reader: Arc<MyNoSqlDataReaderTcp<UserGroupsNoSqlModel>>,
    // ... other dependencies
}
```

Initialize via `ServiceContext::get_ns_reader`:

```rust
use yft_service_sdk::ServiceContext;
use std::sync::Arc;

impl AppContext {
    pub async fn new(sc: &ServiceContext, settings_reader: Arc<SettingsReader>) -> Self {
        let xp_progression_settings_reader = sc
            .get_ns_reader::<XpProgressionSettingsMyNoSqlEntity>()
            .await;

        let user_groups_reader = sc
            .get_ns_reader::<UserGroupsNoSqlModel>()
            .await;

        Self {
            xp_progression_settings_reader,
            user_groups_reader,
            // ... initialize other deps
        }
    }
}
```

## Layer Rules

- **Controllers** — MUST NOT access readers. Map DTOs, call flows, return responses.
- **Flows** — MAY read via `AppContext`. No connection-level details.
- **Repositories** — If needed, wrap reader into a dedicated repo exposed from `AppContext`.

## Reader API

```rust
impl<TMyNoSqlEntity> MyNoSqlDataReaderTcp<TMyNoSqlEntity>
where
    TMyNoSqlEntity: MyNoSqlEntity + MyNoSqlEntitySerializer + Sync + Send + 'static,
{
    pub async fn get_table_snapshot_as_vec(&self) -> Option<Vec<Arc<TMyNoSqlEntity>>>;
    pub async fn get_by_partition_key_as_vec(&self, partition_key: &str) -> Option<Vec<Arc<TMyNoSqlEntity>>>;
    pub async fn get_entity(&self, partition_key: &str, row_key: &str) -> Option<Arc<TMyNoSqlEntity>>;
    pub async fn has_partition(&self, partition_key: &str) -> bool;
    pub async fn get_partition_keys(&self) -> Vec<String>;
}
```

## Common Patterns

**Full table snapshot (small config tables):**
```rust
let items = app
    .xp_progression_settings_reader
    .get_table_snapshot_as_vec()
    .await
    .unwrap_or_default();
```

**Get by partition key:**
```rust
let items = app
    .user_groups_reader
    .get_by_partition_key_as_vec(partition_key)
    .await
    .unwrap_or_default();
```

**Get single entity:**
```rust
if let Some(entity) = app
    .user_groups_reader
    .get_entity(partition_key, row_key)
    .await
{
    // use entity
}
```

**Check partition exists:**
```rust
let exists = app
    .user_groups_reader
    .has_partition(partition_key)
    .await;
```
