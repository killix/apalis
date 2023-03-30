# apalis [![Build Status](https://github.com/geofmureithi/apalis/actions/workflows/ci.yaml/badge.svg)](https://github.com/geofmureithi/apalis/actions)

apalis is a simple, extensible multithreaded background job processing library for Rust.

## Features

- Simple and predictable job handling model.
- Jobs handlers with a macro free API.
- Take full advantage of the [`tower`] ecosystem of
  middleware, services, and utilities.
- Runtime agnostic - Use tokio, smol etc.
- Optional Web interface to help you manage your jobs.

apalis job processing is powered by [`tower::Service`] which means you have access to the [`tower`] middleware.

apalis has support for

- Redis
- SQlite
- PostgresSQL
- MySQL
- Cron Jobs
- Bring Your Own Job Source eg Twitter streams

## Getting Started

To get started, just add to Cargo.toml

```toml
[dependencies]
apalis = { version = "0.4", features = ["redis"] }
```

## Usage

```rust
use apalis::prelude::*;
use apalis::redis::RedisStorage;
use serde::{Deserialize, Serialize};
use anyhow::Result;

#[derive(Debug, Deserialize, Serialize)]
struct Email {
    to: String,
}

async fn email_service(job: Email, ctx: JobContext) {
    info!("Do something");
}

#[tokio::main]
async fn main() -> Result<()> {
    std::env::set_var("RUST_LOG", "debug");
    env_logger::init();
    let redis = std::env::var("REDIS_URL").expect("Missing env variable REDIS_URL");
    let storage = RedisStorage::new(redis).await?;
    Monitor::new()
        .register_with_count(2, move || {
            WorkerBuilder::new("email-worker-1")
                .with_storage(storage.clone())
                .build_fn(email_service)
        })
        .run()
        .await
}

```

Then

```rust
//This can be in another part of the program or another application
async fn produce_route_jobs(storage: &RedisStorage<Email>) -> Result<()> {
    let mut storage = storage.clone();
    storage
        .push(Email {
            to: "test@example.com".to_string(),
        })
        .await?;
}

```

### Web UI

If you are running [apalis Board](https://github.com/geofmureithi/apalis-board), you can easily manage your jobs. See a working [Rest API here](https://github.com/geofmureithi/apalis/tree/master/examples/rest-api)

![UI](https://github.com/geofmureithi/apalis-board/raw/master/screenshots/workers.png)

## Feature flags

- _tracing_ (enabled by default) — Support Tracing 👀
- _redis_ — Include redis storage
- _postgres_ — Include Postgres storage
- _sqlite_ — Include SQlite storage
- _mysql_ — Include MySql storage
- _cron_ — Include cron job processing
- _sentry_ — Support for Sentry exception and performance monitoring
- _prometheus_ — Support Prometheus metrics
- _retry_ — Support direct retrying jobs
- _timeout_ — Support timeouts on jobs
- _limit_ — 💪 Limit the amount of jobs
- _filter_ — Support filtering jobs based on a predicate
- _extensions_ — Add a global extensions to jobs

## Storage Comparison

Since we provide a few storage solutions, here is a table comparing them:

| Feature         | Redis | Sqlite | Postgres | Sled | Mysql | Mongo | Cron |
| :-------------- | :---: | :----: | :------: | :--: | :---: | :---: | :--: |
| Scheduled jobs  |   ✓   |   ✓    |    ✓     |  x   |   ✓   |   x   |  ✓   |
| Retry jobs      |   ✓   |   ✓    |    ✓     |  x   |   ✓   |   x   |  ✓   |
| Persistence     |   ✓   |   ✓    |    ✓     |  x   |   ✓   |   x   | BYO  |
| Rerun Dead jobs |   ✓   |   ✓    |    ✓     |  x   |   ✓   |   x   |  x   |

## How apalis works (Draft)

```mermaid
sequenceDiagram
    participant App
    participant Worker
    participant Postgres

    App->>+Worker: Add job to queue
    Worker->>+Postgres: Poll queue for job
    Postgres-->>-Worker: Job data
    Worker->>+Postgres: Update job status to 'running'
    Postgres-->>-Worker: Confirmation
    Worker->>+App: Notify job started
    loop job execution
        Worker-->>-App: Report job progress
    end
    Worker->>+Postgres: Update job status to 'completed'
    Postgres-->>-Worker: Confirmation
    Worker->>+App: Notify job completed

```

## Thanks to

- [`tower`] - Tower is a library of modular and reusable components for building robust networking clients and servers.
- [redis-rs](https://github.com/mitsuhiko/redis-rs) - Redis library for rust
- [sqlx](https://github.com/launchbadge/sqlx) - The Rust SQL Toolkit

## Roadmap

v 0.4

- [x] Move from actor based to layer based processing
- [x] Graceful Shutdown
- [x] Allow other types of executors apart from Tokio
- [x] Mock/Test Worker
- [x] Improve monitoring
- [ ] Improve apalis Board
- [x] Add job progress via layer
- [x] Add more sources

v 0.3

- [x] Standardize API (Storage, Worker, Data, Middleware, Context )
- [x] Introduce SQL
- [x] Implement layers for Sentry and Tracing.
- [x] Improve documentation
- [x] Organized modules and features.
- [x] Basic Web API Interface
- [x] Sql Examples
- [x] Sqlx migrations

v 0.2

- [x] Redis Example
- [x] Actix Web Example

## Resources

- [Background job processing with rust using actix and redis](https://mureithi.me/blog/background-job-processing-with-rust-actix-redis)

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct, and the process for submitting pull requests to us.

## Versioning

We use [SemVer](http://semver.org/) for versioning. For the versions available, see the [tags on this repository](https://github.com/geofmureithi/apalis/tags).

## Authors

- **Njuguna Mureithi** - _Initial work_ - [Njuguna Mureithi](https://github.com/geofmureithi)

See also the list of [contributors](https://github.com/geofmureithi/apalis/contributors) who participated in this project.

It was formerly `actix-redis-jobs` and if you want to use the crate name please contact me.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details

## Acknowledgments

- Inspiration: The redis part of this project is heavily inspired by [Curlyq](https://github.com/mcmathja/curlyq) which is written in GoLang

[`tower::service`]: https://docs.rs/tower/latest/tower/trait.Service.html
[`tower`]: https://crates.io/crates/tower
[`actix`]: https://crates.io/crates/actix
[`tower-http`]: https://crates.io/crates/tower-http
[`actor`]: https://docs.rs/actix/0.13.0/actix/trait.Actor.html
