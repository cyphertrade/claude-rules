---
paths:
  - "src/background/**/*"
  - "src/**/*publisher*"
  - "src/**/*subscriber*"
---

# MyServiceBus (Publishers & Subscribers)

## Constraints

- **MUST** create publishers via `ServiceContext` helpers — NEVER manually
- **MUST** store publishers in `AppContext` (or services/repositories on `AppContext`)
- **MUST** use shared message model types from existing crates — NEVER define models locally
- **MUST** register subscribers **before** `start_application().await`
- Controllers **MUST NOT** create publishers or access bus directly
- Subscriber `handle_messages` should be a thin adapter — delegate business logic to flows
- If `TopicQueueType` is not obvious from existing code, **ASK the developer** — do not guess

## Setup

Enable the feature in `Cargo.toml`:

```toml
[dependencies]
yft-service-sdk = { path = "yft-service-sdk", features = ["my-service-bus"] }
```

Add the TCP setting to `src/settings.rs`:

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
    pub my_sb_tcp_host_port: String,
}
```

## Publishers

Two types available:

- `MyServiceBusPublisher` — sends once, returns error on failure (caller handles retries)
- `PublisherWithInternalQueue` — buffers and retries in background (fire-and-forget)

`TModel` must implement `MySbMessageSerializer + GetMySbModelTopicId`.

### Wiring publishers into AppContext

```rust
use my_service_bus_abstractions::{GetMySbModelTopicId, MySbMessageSerializer};
use yft_service_sdk::external::my_service_bus::{MyServiceBusPublisher, PublisherWithInternalQueue};

#[derive(Clone)]
pub struct AppContext {
    pub bidask_publisher: MyServiceBusPublisher<BidAskSbModel>,
    pub bidask_publisher_with_queue: PublisherWithInternalQueue<BidAskSbModel>,
    // ... other fields
}
```

```rust
impl AppContext {
    pub async fn new(
        sc: &ServiceContext,
        settings_reader: Arc<SettingsReader>,
    ) -> Self {
        let bidask_publisher =
            sc.get_sb_publisher::<BidAskSbModel>(/* do_retries: */ false).await;

        let bidask_publisher_with_queue =
            sc.get_sb_publisher_with_queue::<BidAskSbModel>().await;

        Self {
            bidask_publisher,
            bidask_publisher_with_queue,
            // ... other dependencies
        }
    }
}
```

### Publishing in flows

```rust
pub async fn publish_bidask_update(
    app: &AppContext,
    model: BidAskSbModel,
) -> Result<(), PublishError> {
    app.bidask_publisher
        .publish(model)
        .await
        .map_err(PublishError::from)
}
```

## Subscribers

Place subscriber files under `src/background/sb/<name>_subscriber.rs`.

### Subscriber struct

```rust
use std::sync::Arc;
use crate::app::AppContext;

pub struct BidAskSbSubscriber {
    pub app: Arc<AppContext>,
}

impl BidAskSbSubscriber {
    pub fn new(app: Arc<AppContext>) -> Self {
        Self { app }
    }
}
```

### SubscriberCallback trait

`TMessageModel` must implement `MySbMessageDeserializer<Item = TMessageModel> + Send + Sync + 'static`.

```rust
#[async_trait::async_trait]
pub trait SubscriberCallback<
    TMessageModel: MySbMessageDeserializer<Item = TMessageModel> + Send + Sync + 'static,
>
{
    async fn handle_messages(
        &self,
        messages_reader: &mut MessagesReader<TMessageModel>,
    ) -> Result<(), MySbSubscriberHandleError>;
}
```

### Implementation example

```rust
use my_service_bus_abstractions::MySbMessageDeserializer;
use my_service_bus_extensions::{MessagesReader, MySbSubscriberHandleError};
use crate::background::sb::bidask_subscriber::BidAskSbSubscriber;
use crate::sb_models::BidAskSbModel;

#[async_trait::async_trait]
impl SubscriberCallback<BidAskSbModel> for BidAskSbSubscriber {
    async fn handle_messages(
        &self,
        messages_reader: &mut MessagesReader<BidAskSbModel>,
    ) -> Result<(), MySbSubscriberHandleError> {
        while let Some(messages) = messages_reader.get_all() {
            let messages = messages
                .into_iter()
                .filter_map(|x| {
                    let message = x.take_message();

                    let bidask_date = /* parse date from message */;

                    let Some(bidask_date) = bidask_date else {
                        tracing::error!(?message, "Failed to parse bidask date. Skipping");
                        return None;
                    };

                    Some(BidAsk {
                        asset_pair: message.id,
                        bid: message.bid,
                        ask: message.ask,
                        base: message.base,
                        quote: message.quote,
                        date: bidask_date,
                    })
                })
                .collect::<Vec<_>>();

            // Delegate to a flow
            handle_bidask_updates(self.app.as_ref(), messages).await;
        }

        Ok(())
    }
}
```

### Registering in main.rs

Register **before** `start_application()`:

```rust
use std::sync::Arc;
use yft_service_sdk::external::my_service_bus::TopicQueueType;
use crate::background::sb::bidask_subscriber::BidAskSbSubscriber;

#[tokio::main]
async fn main() {
    let settings_reader = SettingsReader::new(".my-cfd-platform").await;
    let settings_reader = Arc::new(settings_reader);

    let mut service_context = ServiceContext::new(settings_reader.clone()).await;
    let app_context = Arc::new(AppContext::new(&service_context, settings_reader.clone()).await);

    service_context
        .register_sb_subscribe(
            Arc::new(BidAskSbSubscriber::new(app_context.clone())),
            TopicQueueType::PermanentWithSingleConnection,
        )
        .await;

    service_context.start_application().await;
}
```

## TopicQueueType Decision Tree

```rust
pub enum TopicQueueType {
    Permanent = 0,
    DeleteOnDisconnect = 1,
    PermanentWithSingleConnection = 2,
}
```

- If queue should persist and support multiple consumers → `Permanent`
- If queue should be removed on disconnect (ephemeral) → `DeleteOnDisconnect`
- If queue should persist but only one consumer connection → `PermanentWithSingleConnection`
- If unclear → **ask the developer**

## Layer Rules

- **SettingsModel** — provides `my_sb_tcp_host_port`. No hard-coded connection details.
- **ServiceContext** — owns the bus client. Use `get_sb_publisher`, `get_sb_publisher_with_queue`, `register_sb_subscribe`.
- **AppContext** — stores publishers. Exposes them to flows and background workers.
- **Publishers** — used from flows/repositories to emit messages. Never created in controllers.
- **Subscribers** — implement `SubscriberCallback`, convert bus messages to DTOs, call flows. Registered in `main.rs`.
