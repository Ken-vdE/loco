+++
title = "Multiple Db"
description = ""
date = 2024-03-01T18:10:00+00:00
updated = 2024-03-01T18:10:00+00:00
draft = false
weight = 31
sort_by = "weight"
template = "docs/page.html"

[extra]
lead = ""
toc = true
top = false
flair =[]
+++

`Loco` enables you to work with more than one database and share instances across your application.

To set up an additional database, begin with database connections and configuration. The recommended approach is to navigate to your configuration file and add the following under [settings](@/docs/getting-started/config.md#settings):

```yaml
settings:
  secondary_db:
    uri: {{get_env(name="DATABASE_URL", default="postgres://loco:loco@localhost:5432/secondary_db")}}
    enable_logging: false
    connect_timeout: 500
    idle_timeout: 500
    min_connections: 1
    max_connections: 1
    auto_migrate: true
    dangerously_truncate: false
    dangerously_recreate: false
```


After configuring the database, import [loco-extras](https://crates.io/crates/loco-extras) and enable the `layer-db` feature in your Cargo.toml:
```toml
loco-extras = { version = "*", features = ["layer-db"] }
```


Next, in your app.rs hooks, implement the `after_routes` function as shown below:

```rs
async fn after_routes(router: axum::Router, _ctx: &AppContext) -> Result<axum::Router> {
    let secondary_db_config = _ctx
        .config
        .settings
        .clone()
        .ok_or_eyre("settings config not configured")?;

    let secondary_db = secondary_db_config
        .get("secondary_db")
        .ok_or_eyre("secondary_db not configured")?;
    let router =
        loco_extras::layers::db::add(router, serde_json::from_value(secondary_db.clone())?)
            .await?;

    Ok(router)
}
```

Now, you can use the secondary database in your controller:

```rust
use sea_orm::DatabaseConnection;
use axum::{response::IntoResponse, Extension};

pub async fn list(
    State(ctx): State<AppContext>,
    Extension(secondary_db): Extension<DatabaseConnection>,
) -> Result<impl IntoResponse> {
  let res = Entity::find().all(&secondary_db).await;
}
```

# Many Database Connections

To connect more than two different databases, load the feature `layer-multi-db` in [loco-extras](https://crates.io/crates/loco-extras):
```toml
loco-extras = { version = "*", features = ["loco-extras"] }
```

The database configuration should look like this:
```yaml
settings:
  multi_db: 
    secondary_db:      
      uri: {{get_env(name="DATABASE_URL", default="postgres://loco:loco@localhost:5432/secondary_db")}}      
      enable_logging: false      
      connect_timeout: 500      
      idle_timeout: 500      
      min_connections: 1      
      max_connections: 1      
      auto_migrate: true      
      dangerously_truncate: false      
      dangerously_recreate: false
    third_db:      
      uri: {{get_env(name="DATABASE_URL", default="postgres://loco:loco@localhost:5432/third_db")}}      
      enable_logging: false      
      connect_timeout: 500      
      idle_timeout: 500      
      min_connections: 1      
      max_connections: 1      
      auto_migrate: true      
      dangerously_truncate: false      
      dangerously_recreate: false
```

In the `app.rs` hooks, implement the `after_routes` function for multiple databases:
```rs
async fn after_routes(router: axum::Router, _ctx: &AppContext) -> Result<axum::Router> {
    let secondary_db_config = _ctx
        .config
        .settings
        .clone()
        .ok_or_eyre("settings config not configured")?;

    let multi_db = secondary_db_config
        .get("multi_db")
        .ok_or_eyre("multi_db not configured")?;

    let res = serde_json::from_value(multi_db.clone())?;

    let router = loco_extras::layers::multi_db::add(router, res).await?;

    Ok(router)
}
```

Now, you can use the multiple databases in your controller:

```rust
use sea_orm::DatabaseConnection;
use axum::{response::IntoResponse, Extension};
use loco_rs::db::MultiDb;

pub async fn list(
    State(ctx): State<AppContext>,
    Extension(multi_db): Extension<MultiDb>,
) -> Result<impl IntoResponse> {
  let third_db = multi_db.get("third_db")?;
  let res = Entity::find().all(third_db).await;
}
```