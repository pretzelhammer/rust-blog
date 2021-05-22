
# RESTful API in Sync & Async Rust

_11 May 2021 · #rust · #diesel · #rocket · #sqlx · #actix-web_

**Table of Contents**

- [Intro](#intro)
- [General](#general)
    - [Project Setup](#project-setup)
    - [Loading Environment Variables w/dotenv](#loading-environment-variables-wdotenv)
    - [Handling Dates & Times w/chrono](#handling-dates--times-wchrono)
    - [Logging w/fern](#logging-wfern)
    - [JSON Serialization w/serde](#json-serialization-wserde)
    - [Domain Modeling](#domain-modeling)
- [Sync Implementation](#sync-implementation)
    - [SQL Schema Migrations w/diesel-cli](#sql-schema-migrations-wdiesel-cli)
    - [Executing SQL Queries w/Diesel](#executing-sql-queries-wdiesel)
        - [Mapping DB Enums to Rust Enums](#mapping-db-enums-to-rust-enums)
        - [Fetching Data](#fetching-data)
        - [Inserting Data](#inserting-data)
        - [Updating Data](#updating-data)
        - [Deleting Data](#deleting-data)
        - [Using a Connection Pool w/r2d2](#using-a-connection-pool-wr2d2)
        - [Refactoring DB Operations Into a Module](#refactoring-db-operations-into-a-module)
    - [HTTP Routing w/Rocket](#http-routing-wrocket)
        - [Routing Basics](#routing-basics)
        - [GET Requests](#get-requests)
        - [POST & PATCH Requests](#post--patch-requests)
        - [DELETE Requests](#delete-requests)
        - [Refactoring API Routes Into a Module](#refactoring-api-routes-into-a-module)
        - [Authentication](#authentication)
- [Async Implementation](#async-implementation)
    - [SQL Schema Migrations w/sqlx-cli](#sql-schema-migrations-wsqlx-cli)
    - [Executing SQL Queries w/sqlx](#executing-sql-queries-wsqlx)
        - [Fetching Data](#fetching-data-1)
        - [Inserting Data](#inserting-data-1)
        - [Updating Data](#updating-data-1)
        - [Deleting Data](#deleting-data-1)
        - [Compile-Time Verification of SQL Queries](#compile-time-verification-of-sql-queries)
        - [Using a Connection Pool w/sqlx](#using-a-connection-pool-wsqlx)
        - [Refactoring DB Operations Into a Module](#refactoring-db-operations-into-a-module-1)
    - [HTTP Routing w/actix-web](#http-routing-wactix-web)
        - [Routing Basics](#routing-basics-1)
        - [GET Requests](#get-requests-1)
        - [POST & PATCH Requests](#post--patch-requests-1)
        - [DELETE Requests](#delete-requests-1)
        - [Refactoring API Routes Into a Module](#refactoring-api-routes-into-a-module-1)
        - [Authentication](#authentication-1)
- [Benchmarks](#benchmarks)
    - [Servers](#servers)
    - [Methodology](#methodology)
    - [Calculating Best Possible Performance](#calculating-best-possible-performance)
    - [Measuring Resource Usage](#measuring-resource-usage)
    - [Results](#results)
        - [Read-Only Workload](#read-only-workload)
        - [Reads + Writes Workload](#reads--writes-workload)
- [Concluding Thoughts](#concluding-thoughts)
    - [Diesel vs sqlx](#diesel-vs-sqlx)
    - [Rocket vs actix-web](#rocket-vs-actix-web)
    - [Sync Rust vs Async Rust](#sync-rust-vs-async-rust)
    - [Rust vs JS](#rust-vs-js)
    - [In Summary](#in-summary)
- [Discuss](#discuss)
- [Notifications](#notifications)
- [Further Reading](#further-reading)



## Intro

Let's implement a RESTful API server in Rust for an imaginary Kanban-style project management app. A popular real-world example of such an app is Trello:

![trello board](../assets/trello-board.png)

On its surface Kanban is pretty simple: there's a board and cards. The board represents a project. The cards represent tasks. The position of the cards on the board represents the state and progress of the tasks. The simplest boards have 3 columns for tasks which are: queued (to do), in progress (doing), and done (done).

Despite being simple on its surface, Kanban, and all kinds of project management software in general, is a literal bottomless pit of complexity. There's a million things we could implement, and after we finish the first million things there would be a million more. However, since I'm trying to write a single article and not an entire book series let's keep the feature scope tiny.

The server should support the ability to:
- Create boards
    - Boards have names
- Get a list of all boards
- Delete boards
- Create cards
    - Can associate cards with boards
    - Cards have descriptions and statuses
- Get a list of all the cards on a board
- Get a board summary: count of all the cards on the board grouped by their status
- Update cards
- Delete cards

And that's it! To make this project slightly more interesting let's also include token-based authentication for all of the server's endpoints, but let's keep it simple: as long as a request contains a valid token it has access to all of the boards and cards.

Furthermore, to satisfy my own cursiority, and to maximize the educationalness of this article, we're going to write two implementations together: one using sync Rust and the other using async Rust. The first implementation will use r2d2, Diesel, and Rocket. The second implementation will use sqlx, and actix-web. Here's a quick preview of the crates we'll be using for this project:

General crates
- dotenv (loading environment variables)
- log + fern (logging)
- chrono (date & time handling)
- serde + serde_json (JSON de/serialization)

Sync crates
- diesel-cli (DB schema migrations)
- diesel + diesel-derive-enum (ORM / building SQL queries)
- r2d2 (DB connection pool)
- rocket + rocket_contrib (HTTP routing)

Async crates
- sqlx-cli (DB schema migrations)
- sqlx (executing SQL queries & DB connection pool)
- actix-web (HTTP routing)
- futures (general future-related utilities)

After finishing both sync and async implementations we'll run some benchmarks to see which has better performance, because everyone loves benchmarks.



## General



### Project Setup

All of the boring instructions for setting this project up, like installing Docker and running PostgresQL locally, are in the [companion code repository](https://github.com/pretzelhammer/kanban). For this article let's focus entirely on the fun part: the Rust!

After the initial setup we have this empty `Cargo.toml` file:

```toml
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"
```

And this empty main file:

```rust
// src/main.rs

fn main() {
    println!("Hello, world!");
}
```


### Loading Environment Variables w/dotenv

crates
- dotenv

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

+[dependencies]
+dotenv = "0.15"
```

This crate does one small simple job: it loads variables from an `.env` in the current working directory and adds them to the program's environment variables. Here's the general `.env` file we'll be using:

```bash
# .env

LOG_LEVEL=INFO
LOG_FILE=server.log
DATABASE_URL=postgres://postgres@localhost:5432/postgres 
```

Updated main file which uses `dotenv`:

```rust
// src/main.rs

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    // loads env variables from .env
    dotenv::dotenv()?;

    // example
    assert_eq!("INFO", std::env::var("LOG_LEVEL").unwrap());

    Ok(())
}
```



### Handling Dates & Times w/chrono

crates
- chrono

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
+chrono = "0.4"
```

Rust's go-to library for handling dates & times is chrono. We're not using the dependency in our project _just yet_ but will very soon after we add a few more dependencies.



### Logging w/fern

crates
- log
- fern

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = "0.4"
+log = "0.4"
+fern = "0.6"
```

Log is Rust's logging facade library. It provides the high-level logging API but we still need to pick an implementation, and the implementation we're going to use is the fern crate. Fern allows us to easily customize the logging format and also chain multiple outputs so we can log to stderr and a file if we wanted to. After adding log and fern let's encapsulate all of the logging configuration and initialization into its own module:

```rust
// src/logger.rs

use std::env;
use std::fs;
use log::{debug, error, info, trace, warn};

pub fn init() -> Result<(), fern::InitError> {
    // pull log level from env
    let log_level = env::var("LOG_LEVEL").unwrap_or("INFO".into());
    let log_level = log_level
        .parse::<log::LevelFilter>()
        .unwrap_or(log::LevelFilter::Info);

    let mut builder = fern::Dispatch::new()
        .format(|out, message, record| {
            out.finish(format_args!(
                "[{}][{}][{}] {}",
                chrono::Local::now().format("%H:%M:%S"),
                record.target(),
                record.level(),
                message
            ))
        })
        .level(log_level)
        // log to stderr
        .chain(std::io::stderr());

    // also log to file if one is provided via env
    if let Ok(log_file) = env::var("LOG_FILE") {
        let log_file = fs::File::create(log_file)?;
        builder = builder.chain(log_file);
    }

    // globally apply logger
    builder.apply()?;

    trace!("TRACE output enabled");
    debug!("DEBUG output enabled");
    info!("INFO output enabled");
    warn!("WARN output enabled");
    error!("ERROR output enabled");

    Ok(())
}
```

And then add that module to our main file:

```diff
// src/main.rs

+mod logger;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
+   logger::init()?;

    Ok(())
}
```

If we run the program now, since `INFO` is the default logging level, here's what we'd see:

```bash
$ cargo run
[08:36:30][kanban::logger][INFO] INFO output enabled
[08:36:30][kanban::logger][WARN] WARN output enabled
[08:36:30][kanban::logger][ERROR] ERROR output enabled
```


### JSON Serialization w/serde

crates
- serde
- serde_json

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
- chrono = "0.4"
+ chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
+ serde = { version = "1.0", features = ["derive"] }
+ serde_json = "1.0"
```

Pro-tip: when adding a new dependency to a project it's good to look through existing dependencies to see if they have the new dependency as a feature flag. In this case chrono has serde as a feature flag, which if enabled, adds `serde::Serialize` and `serde::Deserialize` impls to all of chrono's types. This will allow us to use chrono types in our own structs later which we will also derive `serde::Serialize` and `serde::Deserialize` impls for.



### Domain Modeling

Okay, let's start modeling our domain. We know we will have boards so:

```rust
#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Board {
    pub id: i64,
    pub name: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

Unpacking the new stuff:
- `#[derive(serde::Serialize)]` derives a `serde::Serialize` impl for `Board` which will allow us to serialize it to JSON using the `serde_json` crate.
- `#[serde(rename_all = "camelCase")]` renames all of the snake_case member identifiers to camelCase when serializing (or vice versa when deserializing). This is because it's a convention to use snake_case names in Rust but JSON is often produced and consumed by JS code and the JS convention is to use camelCase for member identifiers.
- Making `id` an `i64` instead of an `u64` might seem like an odd choice but since we're using PostgresQL as our DB we have to do this because PostgresQL only supports signed integer types.
- A `created_at` member is always useful to have, if for no other reason than to be able to sort entities by chronological order when no better sort order is available.

Okay, let's add cards and statuses:

```rust
#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Card {
    pub id: i64,
    pub board_id: i64,
    pub description: String,
    pub status: Status,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub enum Status {
    Todo,
    Doing,
    Done,
}
```

Since we'd also like to support returning a "board summary" which contains the count of all of the cards on a board grouped by their status here's the model for that:

```rust
#[derive(serde::Serialize)]
pub struct BoardSummary {
    pub todo: i64,
    pub doing: i64,
    pub done: i64,
}
```

When using the API to create a new board users can provide the board name but not its id, since that will be set by the DB, so we need a model for that as well:

```rust
#[derive(serde::Deserialize)]
pub struct CreateBoard {
    pub name: String,
}
```

Likewise users can also create cards. When creating a card let's assume we only want users to provide the new card's description and what board it should be associated with. The new card will get the default todo status and will get its id set by the DB:

```rust
#[derive(serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CreateCard {
    pub board_id: i64,
    pub description: String,
}
```

When updating a card let's assume we want users to only be able to update the description or the status. It would be pretty weird if we allowed them to move cards between boards which is a pretty unusual feature in most project management apps:

```rust
#[derive(serde::Deserialize)]
pub struct UpdateCard {
    pub description: String,
    pub status: Status,
}
```

Throw all of those into their own module and we get:

```rust
// src/models.rs

// for GET requests

#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Board {
    pub id: i64,
    pub name: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub struct Card {
    pub id: i64,
    pub board_id: i64,
    pub description: String,
    pub status: Status,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub enum Status {
    Todo,
    Doing,
    Done,
}

#[derive(serde::Serialize)]
pub struct BoardSummary {
    pub todo: i64,
    pub doing: i64,
    pub done: i64,
}

// for POST requests

#[derive(serde::Deserialize)]
pub struct CreateBoard {
    pub name: String,
}

#[derive(serde::Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CreateCard {
    pub board_id: i64,
    pub description: String,
}

// for PATCH requests

#[derive(serde::Deserialize)]
pub struct UpdateCard {
    pub description: String,
    pub status: Status,
}
```

And the updated main file:

```diff
// src/main.rs

mod logger;
+mod models;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```


## Sync Implementation



### SQL Schema Migrations w/diesel-cli

crates
- diesel-cli

```bash
cargo install diesel_cli
```

If the above command doesn't work at first, it's likely because we don't have all the development libraries for all of diesel-cli's supported databases. Since we're just using PostgresQL, we can make sure the development libraries are installed with these commands:

```bash
# macOS
brew install postgresql

# ubuntu
apt-get install postgresql libpq-dev
```

And then we can tell cargo to only install diesel-cli with support for PostgresQL:

```bash
cargo install diesel_cli --no-default-features --features postgres
```

Once we have diesel-cli installed we can use it to create new migrations and execute pending migrations. diesel-cli figures out which DB to connect to by checking the `DATABASE_URL` environment variable, which it will also load from an `.env` file if one exists in the current working directory.

Assuming the DB is currently running and a `DATABASE_URL` environment variable is present, here's the first diesel-cli command we'd run to bootstrap our project:

```bash
diesel setup
```

With this diesel-cli creates a `migrations` directory where we can generate and write our DB schema migrations. Let's generate our first migration:

```bash
diesel migration generate create_boards
```

This will create a new directory, e.g. `migrations/<year>-<month>-<day>-<time>_create_boards`, with an `up.sql` and `down.sql` which is where we'll write our SQL queries. The "up" query is for creating or modifying our DB schema, in this case creating a boards table:

```sql
-- create_boards up.sql
CREATE TABLE IF NOT EXISTS boards (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT (CURRENT_TIMESTAMP AT TIME ZONE 'utc')
);

-- seed db with some test data for local dev
INSERT INTO boards
(name)
VALUES
('Test board 1'),
('Test board 2'),
('Test board 3');
```

And the "down" query is for reverting the schema changes made in the "up" query, in this case dropping the created boards table:

```sql
-- create_boards down.sql
DROP TABLE IF EXISTS boards;
```

We will also need to store some cards:

```bash
diesel migration generate create_cards
```

The up query for cards:

```sql
-- create_cards up.sql
CREATE TYPE STATUS_ENUM AS ENUM ('todo', 'doing', 'done');

CREATE TABLE IF NOT EXISTS cards (
    id BIGSERIAL PRIMARY KEY,
    board_id BIGINT NOT NULL,
    description TEXT NOT NULL,
    status STATUS_ENUM NOT NULL DEFAULT 'todo',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT (CURRENT_TIMESTAMP AT TIME ZONE 'utc'),
    CONSTRAINT board_fk
        FOREIGN KEY (board_id)
        REFERENCES boards(id)
        ON DELETE CASCADE
);

-- seed db with some test data for local dev
INSERT INTO cards
(board_id, description, status)
VALUES
(1, 'Test card 1', 'todo'),
(1, 'Test card 2', 'doing'),
(1, 'Test card 3', 'done'),
(2, 'Test card 4', 'todo'),
(2, 'Test card 5', 'todo'),
(3, 'Test card 6', 'done'),
(3, 'Test card 7', 'done');
```

And the down query for cards:

```sql
-- create_cards down.sql
DROP TABLE IF EXISTS cards;
```

After writing our migrations we can run them with this command:

```bash
diesel migration run
```

This executes the migrations in chronological order and also writes a Diesel schema file which should look like something like this at this point:

```rust
// src/schema.rs

table! {
    boards (id) {
        id -> Int8,
        name -> Text,
        created_at -> Timestamptz,
    }
}

table! {
    cards (id) {
        id -> Int8,
        board_id -> Int8,
        description -> Text,
        status -> Status_enum,
        created_at -> Timestamptz,
    }
}

joinable!(cards -> boards (board_id));

allow_tables_to_appear_in_same_query!(
    boards,
    cards,
);
```

The above file will always be generated by diesel-cli and is not something we should ever try to edit by hand, however the `diesel setup` command from earlier also generates a `diesel.toml` configuration file which we can edit if we need to configure or modify how the Diesel schema is generated:

```toml
# diesel.toml

[print_schema]
file = "src/schema.rs"
```

This will be useful to know for later.



### Executing SQL Queries w/Diesel

crates
- diesel

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
+diesel = { version = "1.4", features = ["postgres", "chrono"] }
```

We enable the postgres and chrono feature flags since we'll be connecting to PostgresQL and deserializing PostgresQL's timestamp types to chrono types. Updated main file:

```diff
// src/main.rs

+#[macro_use]
+extern crate diesel;

mod logger;
mod models;
+mod schema;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```

Before decorating our models with Diesel's derive macros we have to bring the generated types from the Diesel schema file into scope:

```diff
// src/models.rs

+use crate::schema::*;

// etc
```

However, if we currently try to run `cargo check` we'd get this:

```none
error[E0412]: cannot find type `Status_enum` in this scope
  --> src/schema.rs:14:19
   |
14 |         status -> Status_enum,
   |                   ^^^^^^^^^^^ not found in this scope
```

Uh oh, now what?



#### Mapping DB Enums to Rust Enums

As you may recall from the preceding section we defined an enum type in PostgresQL like so:

```sql
CREATE TYPE STATUS_ENUM AS ENUM ('todo', 'doing', 'done');
```

Unfortunately Diesel does not support mapping DB enums to Rust enums out-of-the-box in a convenient way, so we have to pull in an unofficial 3rd-party library to do this for us:

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
diesel = { version = "1.4", features = ["postgres", "chrono"] }
+diesel-derive-enum = { version = "1.1", features = ["postgres"] }
```

Then we decorate the enum type with the following derive macro and attribute:

```diff
-#[derive(serde::Serialize, serde::Deserialize)]
+#[derive(serde::Serialize, serde::Deserialize, diesel_derive_enum::DbEnum)]
#[serde(rename_all = "camelCase")]
+#[DieselType = "Status_enum"]
pub enum Status {
    Todo,
    Doing,
    Done,
}
```

The derive macro generates a new type called `Status_enum` within the same scope, and this type can be used by Diesel to map the `STATUS_ENUM` DB enum to our `Status` Rust enum.

Then we update `diesel.toml` with this:

```diff
[print_schema]
file = "src/schema.rs"
+import_types = ["diesel::sql_types::*", "crate::models::Status_enum"]
```

Then we re-generate the Diesel schema file using this command:

```bash
diesel print-schema > ./src/schema.rs
```

Which updates our Diesel schema file to this:

```diff
// src/schema.rs

table! {
+    use diesel::sql_types::*;
+    use crate::models::Status_enum;

    boards (id) {
        id -> Int8,
        name -> Text,
        created_at -> Timestamptz,
    }
}

table! {
+    use diesel::sql_types::*;
+    use crate::models::Status_enum;

    cards (id) {
        id -> Int8,
        board_id -> Int8,
        description -> Text,
        status -> Status_enum,
        created_at -> Timestamptz,
    }
}

joinable!(cards -> boards (board_id));

allow_tables_to_appear_in_same_query!(
    boards,
    cards,
);
```

And now when we run `cargo check` everything type checks just fine. Wow, that was quite the annoying hassle, but thankfully it's over.



#### Fetching Data

Alright, now let's impl some Diesel traits for our models by using the handy derive macros:

```diff
-#[derive(serde::Serialize)]
+#[derive(serde::Serialize, diesel::Queryable)]
#[serde(rename_all = "camelCase")]
pub struct Board {
    pub id: i64,
    pub name: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

Deriving `diesel::Queryable` for a struct allows it to be returned as the result of a Diesel query. The order of the struct's members must match the order of the columns in the rows returned from the Diesel query.

The schema file generated by Diesel exports a DSL we can use to craft DB queries with. One of the major benefits of using the DSL is that it gives us verification at compile-time that all of our SQL queries are syntatically and semantically correct! Here's some commented examples:

```rust
use diesel::prelude::*;
use diesel::PgConnection;

// generated by diesel-cli from DB schema
use crate::schema::boards;

// handwritten by us
use crate::models::Board;

// example of connecting to PostgresQL
fn get_connection() -> PgConnection {
    dotenv::dotenv().unwrap();
    let db_url = env::var("DATABASE_URL").unwrap();
    PgConnection::establish(&db_url).unwrap()
}

// fetches all boards from the boards table,
// diesel can map the DB rows to the Board model
// because we derived Queryable for it and the
// DB row data matches the Board's members
fn all_boards(conn: &PgConnection) -> Vec<Board> {
    boards::table
        .load(conn)
        .unwrap()
}

// fetching boards in chronological order
fn all_boards_chronological(conn: &PgConnection) -> Vec<Board> {
    boards::table
        .order_by(boards::created_at.asc())
        .load(conn)
        .unwrap()
}

// fetching a board by its id
fn board_by_id(conn: &PgConnection, board_id: i64) -> Board {
    boards::table
        .filter(boards::id.eq(board_id))
        .first(conn)
        .unwrap()
}

// fetching boards by their exact name
fn board_by_name(conn: &PgConnection, board_name: &str) -> Vec<Board> {
    boards::table
        .filter(boards::name.eq(board_name))
        .load(conn)
        .unwrap()
}

// fetching boards if their names contain some string
fn board_name_contains(conn: &PgConnection, contains: &str) -> Vec<Board> {
    // in LIKE queries "%" means "match zero or more of any character"
    let contains = format!("%{}%", contains);
    boards::table
        .filter(boards::name.ilike(contains))
        .load(conn)
        .unwrap()
}

// fetching boards created within the past 24 hours
fn recent_boards(conn: &PgConnection) -> Vec<Board> {
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    boards::table
        .filter(boards::created_at.ge(past_day))
        .load(conn)
        .unwrap()
}

// fetching boards create within the past 24 hours
// AND whose name contains some string
// by chaining filter methods
fn recent_boards_and_name_contains(conn: &PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    boards::table
        .filter(boards::name.ilike(contains))
        .filter(boards::created_at.ge(past_day))
        .load(conn)
        .unwrap()
}

// fetching boards create within the past 24 hours
// AND whose name contains some string
// by composing predicate expressions
fn recent_boards_and_name_contains_2(conn: &PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    let predicate = boards::name.ilike(contains).and(boards::created_at.ge(past_day));
    boards::table
        .filter(predicate)
        .load(conn)
        .unwrap()
}

// fetching boards create within the past 24 hours
// OR whose name contains some string
// by chaining filter methods
fn recent_boards_or_name_contains(conn: &PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    boards::table
        .filter(boards::name.ilike(contains))
        .or_filter(boards::created_at.ge(past_day))
        .load(conn)
        .unwrap()
}

// fetching boards create within the past 24 hours
// OR whose name contains some string
// by composing predicate expressions
fn recent_boards_or_name_contains_2(conn: &PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    let past_day = chrono::Utc::now() - chrono::Duration::days(1);
    let predicate = boards::name.ilike(contains).or(boards::created_at.ge(past_day));
    boards::table
        .filter(predicate)
        .load(conn)
        .unwrap()
}
```

Okay, let's also derive `diesel::Queryable` for cards:

```diff
-#[derive(serde::Serialize)]
+#[derive(serde::Serialize, diesel::Queryable)]
#[serde(rename_all = "camelCase")]
pub struct Card {
    pub id: i64,
    pub board_id: i64,
    pub description: String,
    pub status: Status,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

We went over a bunch of query examples above but here's a few more:

```rust
use diesel::prelude::*;
use diesel::PgConnection;

// generated by diesel-cli from DB schema
use crate::schema::cards;

// handwritten by us
use crate::models::{Card, Status};

// fetch all cards
fn all_cards(conn: &PgConnection) -> Vec<Card> {
    cards::table
        .load(conn)
        .unwrap()
}

// fetch cards by board
fn cards_by_board(conn: &PgConnection, board_id: i64) -> Vec<Card> {
    cards::table
        .filter(cards::board_id.eq(board_id))
        .load(conn)
        .unwrap()
}

// fetch cards by status
fn cards_by_status(conn: &PgConnection, status: Status) -> Vec<Card> {
    cards::table
        .filter(cards::status.eq(status))
        .load(conn)
        .unwrap()
}
```

So it seems like we can craft almost every query our server needs to support using Diesel's DSL. _Almost_. One of the queries we'd like to support is returning a "board summary" which is the count of all the cards within a board grouped by their status. This is how we'd write the SQL query:

```sql
SELECT count(*), status
FROM cards
WHERE cards.board_id = $1
GROUP BY status;
```

Unfortunately we cannot use Diesel's generated DSL to craft this query. Like all ORMs though, Diesel provides a method to run SQL directly, and that method is `diesel::sql_query` so let's talk about how we'd use that to get our board summary.

First we need to create a new model which will be the result of our query and derive `diesel::QueryableByName` for it as well as decorate its members with the diesel types they map to:

```rust
#[derive(Default, serde::Serialize)]
pub struct BoardSummary {
    pub todo: i64,
    pub doing: i64,
    pub done: i64,
}

// this will be the result of our diesel::sql_query query
#[derive(diesel::QueryableByName)]
pub struct StatusCount {
    #[sql_type = "diesel::sql_types::BigInt"]
    pub count: i64,
    #[sql_type = "Status_enum"]
    pub status: Status,
}

// converting from a list of StatusCounts to a BoardSummary
impl From<Vec<StatusCount>> for BoardSummary {
    fn from(counts: Vec<StatusCount>) -> BoardSummary {
        let mut summary = BoardSummary::default();
        for StatusCount { count, status } in counts {
            match status {
                Status::Todo => summary.todo += count,
                Status::Doing => summary.doing += count,
                Status::Done => summary.done += count,
            }
        }
        summary
    }
}
```

Why do structs which are the result of "regular" Diesel queries have to derive `diesel::Queryable` but structs which are the result of `diesel::sql_query` queries have to derive `diesel::QueryableByName`? According to the Diesel docs the former deserializes DB row columns by index and the latter deserializes DB row columns by name, and the first one is more performant but the latter is a bit more foolproof, hence its use for arbitrary SQL queries.

Anyway, after jumping through all of the hoops above we can finally implement a function to fetch board summaries:

```rust
// fetching summary of cards on a board by id
fn board_summary(conn: &PgConnection, board_id: i64) -> BoardSummary {
    diesel::sql_query(format!(
        "SELECT count(*), status FROM cards WHERE cards.board_id = {} GROUP BY status",
        board_id
    ))
    .load::<StatusCount>(conn)
    .unwrap()
    .into()
}
```



#### Inserting Data

We don't want to just fetch boards, we'd like to create them as well! As a reminder, the `CreateBoard` model only has a `name` member because the `id` and `created_at` members will be set by the DB. And this is how we'd update our `CreateBoard` model to use it with Diesel's DSL:

```diff
// boards has to be in scope to be used in table_name attribute
use crate::schema::boards;

-#[derive(serde::Deserialize)]
+#[derive(serde::Deserialize, diesel::Insertable)]
+#[table_name = "boards"]
pub struct CreateBoard {
    pub name: String,
}
```

After deriving an `diesel::Insertable` impl for `CreateBoard` we can use it directly in Diesel insert queries:

```rust
use diesel::prelude::*;
use diesel::PgConnection;

use crate::schema::boards;
use crate::models::{Board, CreateBoard};

// create and return new board from CreateBoard model
fn create_board(conn: &PgConnection, create_board: CreateBoard) -> Board {
    diesel::insert_into(boards::table)
        .values(&create_board)
        .get_result(conn) // return inserted board result
        .unwrap()
}

// same as above except without using the CreateBoard model
fn create_board_2(conn: &PgConnection, board_name: String) -> Board {
    diesel::insert_into(boards::table)
        .values(boards::name.eq(board_name))
        .get_result(conn) // return inserted board result
        .unwrap()
}
```

We follow the same steps for the `CreateCard` struct:

```diff
// cards has to be in scope to be used in table_name attribute
use crate::schema::cards;

- #[derive(serde::Deserialize)]
+ #[derive(serde::Deserialize, diesel::Insertable)]
#[serde(rename_all = "camelCase")]
+ #[table_name = "cards"]
pub struct CreateCard {
    pub board_id: i64,
    pub description: String,
}
```

Example create functions:

```rust
use diesel::prelude::*;
use diesel::PgConnection;

use crate::schema::cards;
use crate::models::{Card, CreateCard};

// create and return new card using CreateCard model
fn create_card(conn: &PgConnection, create_card: CreateCard) -> Card {
    diesel::insert_into(cards::table)
        .values(&create_card)
        .get_result(conn) // return inserted card result
        .unwrap()
}

// same as above except without using CreateCard model
fn create_card_2(conn: &PgConnection, board_id: i64, description: String) -> Card {
    diesel::insert_into(cards::table)
        .values((cards::board_id.eq(board_id), cards::description.eq(description)))
        .get_result(conn) // returns inserted card result
        .unwrap()
}
```



#### Updating Data

If we derive `diesel::AsChangeSet` for the `UpdateCard` model:

```diff
// cards has to be in scope to be used in table_name attribute
use crate::schema::cards;

-#[derive(serde::Deserialize)]
+#[derive(serde::Deserialize, diesel::AsChangeset)]
+#[table_name = "cards"]
pub struct UpdateCard {
    pub description: String,
    pub status: Status,
}
```

We can now pass `UpdateCard` directly to the `set` method of Diesel update queries:

```rust
use diesel::prelude::*;
use diesel::PgConnection;

use crate::schema::cards;
use crate::models::{Card, UpdateCard};

// update card in DB using UpdateCard model
fn update_card(conn: &PgConnection, card_id: i64, update_card: UpdateCard) -> Card {
    diesel::update(cards::table.filter(cards::id.eq(card_id)))
        .set(update_card)
        .get_result(conn) // return updated card result
        .unwrap()
}

// same as above except without using model
fn update_card_2(conn: &PgConnection, card_id: i64, description: String, status: Status) -> Card {
    diesel::update(cards::table.filter(cards::id.eq(card_id)))
        .set((cards::description.eq(description), cards::status.eq(status)))
        .get_result(conn) // return updated card result
        .unwrap()
}
```



#### Deleting Data

Some delete query examples:

```rust
use diesel::prelude::*;
use diesel::PgConnection;
use crate::schema::{boards, cards};

// delete all boards
fn delete_all_boards(conn: &PgConnection) {
    diesel::delete(boards::table)
        .execute(conn)
        .unwrap();
}

// delete a board by its id
fn delete_board_by_id(conn: &PgConnection, board_id: i64) {
    diesel::delete(boards::table.filter(boards::id.eq(board_id)))
        .execute(conn)
        .unwrap();
}

// delete all cards
fn delete_all_cards(conn: &PgConnection) {
    diesel::delete(cards::table)
        .execute(conn)
        .unwrap();
}

// delete a card by its id
fn delete_card_by_id(conn: &PgConnection, card_id: i64) {
    diesel::delete(cards::table.filter(cards::id.eq(card_id)))
        .execute(conn)
        .unwrap();
}

// delete all the cards on a board
fn delete_cards_by_board(conn: &PgConnection, board_id: i64) {
    diesel::delete(cards::table.filter(cards::board_id.eq(board_id)))
        .execute(conn)
        .unwrap();
}

// delete all done cards on a board
fn delete_done_cards_by_board(conn: &PgConnection, board_id: i64) {
    diesel::delete(
        cards::table
            .filter(cards::board_id.eq(board_id))
            .filter(cards::status.eq(Status::Done)),
    )
    .execute(conn)
    .unwrap();
}
```



#### Using a Connection Pool w/r2d2

crates
- r2d2

Creating DB connections is expensive, so let's use a connection pool library to handle the hard work of managing and reusing DB connections for us. The library we're going to use is r2d2, which comes as a standalone crate but can also can come packaged within Diesel if we enable the r2d2 feature flag, so let's do that:

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
-diesel = { version = "1.4", features = ["postgres", "chrono"] }
+diesel = { version = "1.4", features = ["postgres", "chrono", "r2d2"] }
diesel-derive-enum = { version = "1.1", features = ["postgres"] }
```

Here's how we'd create a pool with the default configuration options:

```rust
use diesel::prelude::*;
use diesel::PgConnection;
use diesel::r2d2;

type PgPool = r2d2::Pool<r2d2::ConnectionManager<PgConnection>>;

fn get_pool() -> PgPool {
    dotenv::dotenv().unwrap();
    let db_url = env::var("DATABASE_URL").unwrap();
    let manager = r2d2::ConnectionManager::new(db_url);
    let pool = r2d2::Pool::new(manager).unwrap();
    pool
}

fn use_pool(pool: &PgPool) {
    let connection = pool.get().unwrap();
    // use connection
}
```



#### Refactoring DB Operations Into a Module

The rest of our application doesn't need to know or care that we're using Diesel and r2d2, so let's abstract all of those implementation details away within a `Db` struct, which we'll put into a `db` module:

```rust
// src/db.rs

use diesel::prelude::*;
use diesel::PgConnection;
use diesel::r2d2;

use crate::StdErr;
use crate::models::*;
use crate::schema::*;

type PgPool = r2d2::Pool<r2d2::ConnectionManager<PgConnection>>;

pub struct Db {
    pool: PgPool,
}

impl Db {
    pub fn connect() -> Result<Self, StdErr> {
        let db_url = env::var("DATABASE_URL")?;
        let manager = r2d2::ConnectionManager::new(db_url);
        let pool = r2d2::Pool::new(manager)?;
        Ok(Db { pool })
    }

    pub fn boards(&self) -> Result<Vec<Board>, StdErr> {
        let conn = self.pool.get()?;
        Ok(boards::table.load(&conn)?)
    }

    pub fn board_summary(&self, board_id: i64) -> Result<BoardSummary, StdErr> {
        let conn = self.pool.get()?;
        let counts: Vec<StatusCount> = diesel::sql_query(format!(
            "select count(*), status from cards where cards.board_id = {} group by status",
            board_id
        ))
        .load(&conn)?;
        Ok(counts.into())
    }

    pub fn create_board(&self, create_board: CreateBoard) -> Result<Board, StdErr> {
        let conn = self.pool.get()?;
        let board = diesel::insert_into(boards::table)
            .values(&create_board)
            .get_result(&conn)?;
        Ok(board)
    }

    pub fn delete_board(&self, board_id: i64) -> Result<(), StdErr> {
        let conn = self.pool.get()?;
        diesel::delete(boards::table.filter(boards::id.eq(board_id))).execute(&conn)?;
        Ok(())
    }

    pub fn cards(&self, board_id: i64) -> Result<Vec<Card>, StdErr> {
        let conn = self.pool.get()?;
        let cards = cards::table
            .filter(cards::board_id.eq(board_id))
            .load(&conn)?;
        Ok(cards)
    }

    pub fn create_card(&self, create_card: CreateCard) -> Result<Card, StdErr> {
        let conn = self.pool.get()?;
        let card = diesel::insert_into(cards::table)
            .values(create_card)
            .get_result(&conn)?;
        Ok(card)
    }

    pub fn update_card(&self, card_id: i64, update_card: UpdateCard) -> Result<Card, StdErr> {
        let conn = self.pool.get()?;
        let card = diesel::update(cards::table.filter(cards::id.eq(card_id)))
            .set(update_card)
            .get_result(&conn)?;
        Ok(card)
    }

    pub fn delete_card(&self, card_id: i64) -> Result<(), StdErr> {
        let conn = self.pool.get()?;
        diesel::delete(
            cards::table.filter(cards::id.eq(card_id)),
        )
        .execute(&conn)?;
        Ok(())
    }
}
```

Updated main file:

```diff
// src/main.rs

#[macro_use]
extern crate diesel;

+mod db;
mod logger;
mod models;
mod schema;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```


### HTTP Routing w/Rocket

crates
- rocket
- rocket_contrib

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
diesel = { version = "1.4", features = ["postgres", "chrono", "r2d2"] }
diesel-derive-enum = { version = "1.1", features = ["postgres"] }
+rocket = "0.4"
+rocket_contrib = "0.4"
```

Rocket v0.4 depends on some nightly features, some of which are only available on compiler version 1.53.0 since they're later removed, so we'll need to install a specific nightly compiler and then switch to using these commands

```bash
# install rustc 1.53.0-nightly (07e0e2ec2 2021-03-24)
rustup install nightly-2021-03-24

# set it as default
rustup default nightly-2021-03-24
```

We also have to add some compiler feature flags to the top of our main:

```diff
// src/main.rs

+// required for rocket macros to work
+#![feature(proc_macro_hygiene, decl_macro)]

#[macro_use]
extern crate diesel;

mod db;
mod logger;
mod models;
mod schema;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```

While we're here let's throw together a quick Rocket hello world:

```diff
// required for rocket macros to work
#![feature(proc_macro_hygiene, decl_macro)]

#[macro_use]
extern crate diesel;

mod db;
mod logger;
mod models;
mod routes;
mod schema;

type StdErr = Box<dyn std::error::Error>;

+#[rocket::get("/")]
+fn hello_world() -> &'static str {
+   "Hello, world!"
+}

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

+   rocket::ignite()
+       .mount("/", rocket::routes![hello_world])
+       .launch();

    Ok(())
}
```

The code above does exactly what you think it does. Also, if we `cargo run` it:

```bash
$ cargo run
[13:07:21][launch][INFO] Configured for development.
[13:07:21][launch_][INFO] address: localhost
[13:07:21][launch_][INFO] port: 8000
[13:07:21][launch_][INFO] log: normal
[13:07:21][launch_][INFO] workers: 32
[13:07:21][launch_][INFO] secret key: generated
[13:07:21][launch_][INFO] limits: forms = 32KiB
[13:07:21][launch_][INFO] keep-alive: 5s
[13:07:21][launch_][INFO] read timeout: 5s
[13:07:21][launch_][INFO] write timeout: 5s
[13:07:21][launch_][INFO] tls: disabled
[13:07:21][rocket::rocket][INFO] Mounting /:
[13:07:21][_][INFO] GET / (hello_world)
[13:07:21][launch][INFO] Rocket has launched from http://localhost:8000
```

Look at all those beautiful logs! I bet you were wondering if, and when, all the logging setup work we did forever ago was going to become useful. Well that time is now! Of course, being good developers who care about being able to debug issues when they arise, we should be logging way more in our application, but I've been intentionally omitting the logging statements because they're noisy and have little educational value. Please imagine that they are there.



#### Routing Basics

We can decorate functions with Rocket's procedural macros to turn them into request handlers, example:

```rust
#[rocket::get("/")]
fn index() -> &'static str {
    "I'm the index route!"
}

#[rocket::get("/nested/route")]
fn index() -> &'static str {
    "I'm some nested route!"
}
```

Data can be extracted from the path using `<param>` placeholders:

```rust
#[rocket::get("/hello/<name>")]
fn greet(name: String) -> String {
    format!("Hello, {}!", name)
}
```

Any type which impls `rocket::request::FromParam` can be extracted from the path:

```rust
use rocket::request::FromParam;

// Rocket provides FromParam impls for
// - all stdlib number types
// - bool
// - String
// - &str
// - some other misc types
#[rocket::get("/echo/<string>/<num>/<maybe>/etc")]
fn echo_path(string: String, num: usize, maybe: bool) -> String {
    format!("got string {}, num {}, and maybe {}", string, num, maybe)
}

// Rocket also provides FromParam impls for
// - Option<T> where T: FromParam
//     - returns Some(T) on success, None otherwise
// - Result<T> where T: FromParam
//     - returns Ok(T) on success, Err(<T as FromParam>::Error) otherwise

// example custom type
struct EvenNumber(i32);

// example FromParam impl
impl<'a> FromParam<'a> for EvenNumber {
    type Error = &'static str;

    fn from_param(param: &'a RawStr) -> Result<Self, Self::Error> {
        let result = i32::from_param(param);
        if let Ok(num) = result {
            if num % 2 == 0 {
                Ok(EvenNumber(num))
            } else {
                Err("param not an even number")
            }
        } else {
            Err("param not a number")
        }
    }
}

// extracting our own custom defined type
#[rocket::get("/<even>")]
fn even_param(even: EvenNumber) -> String {
    format!("got even number {}", even.0)
}
```

A request handler's return type can be anything that impls `rocket::response::Responder`:

```rust
use std::io;
use std::fs::File;
use rocket::{http, request, response};
use rocket::response::{Response, Responder};

// Rocket provides Responder impls for
// - &str
// - String
// - &[u8]
// - Vec<u8>
// - File
// - ()
// - some other misc types
#[rocket::get("/cargo")]
fn returns_cargo() -> File {
    File::open("Cargo.toml").unwrap()
}

// Rocket also provides Responder impls for
// - Option<T> where T: Responder
//     - returns T if Some(T), 404 Not Found if None
// - Result<T, E> where T: Responder, E: Debug
//     - returns T if Ok(T), 500 Internal Server Error if Err(E)

// example custom type
struct EvenNumber(i32);

// example Responder impl
impl<'r> Responder<'r> for EvenNumber {
    fn respond_to(self, _req: &request::Request<'_>) -> response::Result<'r> {
        Response::build()
            .status(http::Status::Ok)
            .raw_header("X-Number-Parity", "Even")
            .sized_body(io::Cursor::new(format!("returning even number {}", self.0)))
            .ok()
    }
}

// returning custom defined type
#[rocket::get("/even")]
fn returns_even() -> EvenNumber {
    EvenNumber(2)
}
```



#### GET Requests

The first problem we need to solve is how to pass our `Db` to our Rocket request handlers. We can achieve this by first passing the `Db` instance to the Rocket application builder via the `manage` method which tells Rocket this is some application state that it should manage for us:

```diff
// src/main.rs

// required for rocket macros to work
#![feature(proc_macro_hygiene, decl_macro)]

#[macro_use]
extern crate diesel;

mod db;
mod logger;
mod models;
mod schema;
mod routes;

type StdErr = Box<dyn std::error::Error>;

#[rocket::get("/")]
fn hello_world() -> &'static str {
    "Hello, world!"
}

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

+   let db = db::Db::connect()?;

    rocket::ignite()
+       .manage(db)
        .mount("/", rocket::routes![hello_world])
        .launch();

    Ok(())
}
```

Then we can retrieve the managed application state from Rocket in our request handlers by wrapping the data type we wish to receive with `rocket::State` as one of our function parameters:

```rust
use rocket::State;
use crate::db::Db;

#[rocket::get("/use/db")]
fn use_db(db: State<Db>) {
    // use db
}
```

We can automatically return JSON by wrapping the return type with the `rocket_contrib::json::Json` type which impls `Responder` and will serialize the wrapped type to JSON using serde and set the response header `Content-Type: application/json`, example:

```rust
use rocket_contrib::json::Json;
use crate::models::Status;

#[rocket::get("/example/json")]
fn return_json() -> Json<Status> {
    Json(Status::Todo)
}
```

Okay, we now have enough context to write all of our GET request handlers:

```rust
use rocket::State;
use rocket_contrib::json::Json;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, Card, BoardSummary};

// GET requests

#[rocket::get("/boards")]
fn boards(db: State<Db>) -> Result<Json<Vec<Board>>, StdErr> {
    db.boards().map(Json)
}

#[rocket::get("/boards/<board_id>/summary")]
fn board_summary(db: State<Db>, board_id: i64) -> Result<Json<BoardSummary>, StdErr> {
    db.board_summary(board_id).map(Json)
}

#[rocket::get("/boards/<board_id>/cards")]
fn cards(db: State<Db>, board_id: i64) -> Result<Json<Vec<Card>>, StdErr> {
    db.cards(board_id).map(Json)
}
```



#### POST & PATCH Requests

`rocket_contrib::json::Json` not only impls `Responder` but also impls `rocket::data::FromData` which means we can use it as a function parameter to a request handler and Rocket will attempt deserialize the request's JSON body into the type we specify, so here's how we'd write the POST & PATCH request handlers:

```rust
use rocket::State;
use rocket_contrib::json::Json;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, CreateBoard, Card, CreateCard, UpdateCard};

// POST requests

#[rocket::post("/boards", data = "<create_board>")]
fn create_board(db: State<Db>, create_board: Json<CreateBoard>) -> Result<Json<Board>, StdErr> {
    db.create_board(create_board.0).map(Json)
}

#[rocket::post("/cards", data = "<create_card>")]
fn create_card(db: State<Db>, create_card: Json<CreateCard>) -> Result<Json<Card>, StdErr> {
    db.create_card(create_card.0).map(Json)
}

// PATCH requests

#[rocket::patch("/cards/<card_id>", data = "<update_card>")]
fn update_card(
    db: State<Db>,
    card_id: i64,
    update_card: Json<UpdateCard>,
) -> Result<Json<Card>, StdErr> {
    db.update_card(card_id, update_card.0).map(Json)
}
```



#### DELETE Requests

The DELETE request handlers:

```rust
use rocket::State;
use rocket_contrib::json::Json;

use crate::StdErr;
use crate::db::Db;

#[rocket::delete("/boards/<board_id>")]
fn delete_board(db: State<Db>, board_id: i64) -> Result<(), StdErr> {
    db.delete_board(board_id)
}

#[rocket::delete("/cards/<card_id>")]
fn delete_card(db: State<Db>, card_id: i64) -> Result<(), StdErr> {
    db.delete_card(card_id)
}
```



#### Refactoring API Routes Into A Module

Let's clean up our code and put all of the API routes into their own module:

```rust
// src/routes.rs

use rocket::http;
use rocket::response;
use rocket::request
use rocket::State;
use rocket_contrib::json::Json;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, CreateBoard, BoardSummary, Card, CreateCard, UpdateCard};

// board routes

#[rocket::get("/boards")]
fn boards(db: State<Db>) -> Result<Json<Vec<Board>>, StdErr> {
    db.boards().map(Json)
}

#[rocket::post("/boards", data = "<create_board>")]
fn create_board(db: State<Db>, create_board: Json<CreateBoard>) -> Result<Json<Board>, StdErr> {
    db.create_board(create_board.0).map(Json)
}

#[rocket::get("/boards/<board_id>/summary")]
fn board_summary(db: State<Db>, board_id: i64) -> Result<Json<BoardSummary>, StdErr> {
    db.board_summary(board_id).map(Json)
}

#[rocket::delete("/boards/<board_id>")]
fn delete_board(db: State<Db>, board_id: i64) -> Result<(), StdErr> {
    db.delete_board(board_id)
}

// card routes

#[rocket::get("/boards/<board_id>/cards")]
fn cards(db: State<Db>, board_id: i64) -> Result<Json<Vec<Card>>, StdErr> {
    db.cards(board_id).map(Json)
}

#[rocket::post("/cards", data = "<create_card>")]
fn create_card(db: State<Db>, create_card: Json<CreateCard>) -> Result<Json<Card>, StdErr> {
    db.create_card(create_card.0).map(Json)
}

#[rocket::patch("/cards/<card_id>", data = "<update_card>")]
fn update_card(
    db: State<Db>,
    card_id: i64,
    update_card: Json<UpdateCard>,
) -> Result<Json<Card>, StdErr> {
    db.update_card(card_id, update_card.0).map(Json)
}

#[rocket::delete("/cards/<card_id>")]
fn delete_card(db: State<Db>, card_id: i64) -> Result<(), StdErr> {
    db.delete_card(card_id)
}

// single public function which returns all API routes

pub fn api() -> Vec<rocket::Route> {
    rocket::routes![
        boards,
        create_board,
        board_summary,
        delete_board,
        cards,
        create_card,
        update_card,
        delete_card,
    ]
}
```

Updated main file:

```diff
// src/main.rs

// required for rocket macros to work
#![feature(proc_macro_hygiene, decl_macro)]

#[macro_use]
extern crate diesel;

mod db;
mod logger;
mod models;
+mod routes;
mod schema;

type StdErr = Box<dyn std::error::Error>;

#[rocket::get("/")]
fn hello_world() -> &'static str {
    "Hello, world!"
}

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    let db = db::Db::connect()?;

    rocket::ignite()
        .manage(db)
        .mount("/", rocket::routes![hello_world])
+       .mount("/api", routes::api())
        .launch();

    Ok(())
}
```

We're almost there! Only one small piece of the puzzle is missing: authentication.



#### Authentication

Let's implement token-based authentication for our RESTful API server. When a request comes in, we'll first check if it has a `Authorization: Bearer <token>` header, and if not reject it immediately. Otherwise, we validate the `<token>` by checking that it exists in a `tokens` DB table, and if it's valid we respond to the request.

Since this is a toy project we're not going to implement different access permissions per token, all tokens will have access to all boards and all cards. Also, let's skip the implementing the handler which generates and returns tokens because it's mostly boilerplate and not relevant to us implementing authentication.

Anyway, we're gonna need to create a `tokens` table first:

```bash
diesel migration generate create_tokens
```

SQL query to create the `tokens` table:

```sql
-- create_tokens up.sql
CREATE TABLE IF NOT EXISTS tokens (
    id TEXT PRIMARY KEY,
    expired_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- seed db with some test data for local dev
INSERT INTO tokens
(id, expired_at)
VALUES
('LET_ME_IN', (CURRENT_TIMESTAMP + INTERVAL '15 minutes') AT TIME ZONE 'utc');
```

SQL query to drop the `tokens` table:

```sql
-- create_tokens down.sql
DROP TABLE IF EXISTS tokens;
```

Then we run the new migration with:

```bash
diesel migration run
```

The above command also has the side-effect of updating the generated Diesel schema file with the new tokens table metadata:

```diff
// src/schema.rs

table! {
    use diesel::sql_types::*;
    use crate::models::Status_enum;

    boards (id) {
        id -> Int8,
        name -> Text,
        created_at -> Timestamptz,
    }
}

table! {
    use diesel::sql_types::*;
    use crate::models::Status_enum;

    cards (id) {
        id -> Int8,
        board_id -> Int8,
        description -> Text,
        status -> Status_enum,
        created_at -> Timestamptz,
    }
}

+table! {
+   use diesel::sql_types::*;
+   use crate::models::Status_enum;
+
+   tokens (id) {
+       id -> Text,
+       expired_at -> Timestamptz,
+   }
+}

joinable!(cards -> boards (board_id));

allow_tables_to_appear_in_same_query!(
    boards,
    cards,
+   tokens,
);
```

Then we derive a `diesel::Queryable` impl for the `Token` model:

```diff
// src/models.rs

use crate::schema::*;

+// for authentication
+
+#[derive(diesel::Queryable)]
+pub struct Token {
+    pub id: String,
+    pub expired_at: chrono::DateTime<chrono::Utc>,
+}

// other models
```

And then add the following method to our `Db` struct to validate tokens:

```diff
use std::env;

use diesel::prelude::*;
use diesel::r2d2;
use diesel::PgConnection;

use crate::models::*;
use crate::schema::*;
use crate::StdErr;

type PgPool = r2d2::Pool<r2d2::ConnectionManager<PgConnection>>;

pub struct Db {
    pool: PgPool,
}

impl Db {
    pub fn connect() -> Result<Self, StdErr> {
        let db_url = env::var("DATABASE_URL")?;
        let manager = r2d2::ConnectionManager::new(db_url);
        let pool = r2d2::Pool::new(manager)?;
        Ok(Db { pool })
    }

+   // token methods
+
+   pub fn validate_token(&self, token_id: &str) -> Result<Token, StdErr> {
+       let conn = self.pool.get()?;
+       let token = tokens::table
+           .filter(tokens::id.eq(token_id))
+           .filter(tokens::expired_at.ge(diesel::dsl::now))
+           .first(&conn)?;
+       Ok(token)
+   }

    // other methods
}
```

We can implement authentication in Rocket in a couple ways, by using either Rocket middleware or Rocket request guards. The official Rocket docs recommend the latter so let's go with that.

A request guard is any type within a handler's parameters which impls `rocket::request::FromRequest`. So all we have to do is just impl that for `Token` and then add `Token` parameters to all our request handlers and we're golden. Here's the `FromRequest` impl on `Token`:

```rust
use rocket::request::{FromRequest, Request, Outcome};
use crate::models::Token;

impl<'a, 'r> FromRequest<'a, 'r> for Token {
    type Error = &'static str;
    fn from_request(request: &'a Request<'r>) -> Outcome<Self, Self::Error> {
        // get request headers
        let headers = request.headers();

        // check that Authorization header exists
        let maybe_auth_header = headers.get_one("Authorization");
        if maybe_auth_header.is_none() {
            return Outcome::Failure((
                http::Status::Unauthorized,
                "missing Authorization header",
            ));
        }

        // and is well-formed
        let auth_header = maybe_auth_header.unwrap();
        let mut auth_header_parts = auth_header.split_ascii_whitespace();
        let maybe_auth_type = auth_header_parts.next();
        if maybe_auth_type.is_none() {
            return Outcome::Failure((
                http::Status::Unauthorized,
                "malformed Authorization header",
            ));
        }

        // and uses the Bearer token authorization method
        let auth_type = maybe_auth_type.unwrap();
        if auth_type != "Bearer" {
            return Outcome::Failure((
                http::Status::BadRequest,
                "invalid Authorization type",
            ));
        }

        // and the Bearer token is present
        let maybe_token_id = auth_header_parts.next();
        if maybe_token_id.is_none() {
            return Outcome::Failure((http::Status::Unauthorized, "missing Bearer token"));
        }
        let token_id = maybe_token_id.unwrap();

        // we can use request.guard::<T>() to get a T from a request
        // which includes managed application state like our Db
        let outcome_db = request.guard::<State<Db>>();
        let db: State<Db> = match outcome_db {
            Outcome::Success(db) => db,
            _ => return Outcome::Failure((http::Status::InternalServerError, "internal error")),
        };

        // validate token
        let token_result = db.validate_token(token_id);
        match token_result {
            Ok(token) => Outcome::Success(token),
            Err(_) => Outcome::Failure((
                http::Status::Unauthorized,
                "invalid or expired Bearer token",
            )),
        }
    }
}
```

Now we add a `Token` parameter to every request handler we want to guard, which is effectively how we implement authentication:

```diff
// src/routes.rs

use rocket::http;
use rocket::State;
use rocket::request::{FromRequest, Request, Outcome};
use rocket_contrib::json::Json;

use crate::db::Db;
use crate::models::*;
use crate::StdErr;

impl<'a, 'r> FromRequest<'a, 'r> for Token {
    type Error = &'static str;
    fn from_request(request: &'a Request<'r>) -> Outcome<Self, Self::Error> {
        // impl
    }
}

// board routes

#[rocket::get("/boards")]
-fn boards(db: State<Db>) -> Result<Json<Vec<Board>>, StdErr> {
+fn boards(db: State<Db>, _t: Token) -> Result<Json<Vec<Board>>, StdErr> {
    db.boards().map(Json)
}

#[rocket::post("/boards", data = "<create_board>")]
-fn create_board(db: State<Db>, create_board: Json<CreateBoard>) -> Result<Json<Board>, StdErr> {
+fn create_board(db: State<Db>, create_board: Json<CreateBoard>, _t: Token) -> Result<Json<Board>, StdErr> {
    db.create_board(create_board.0).map(Json)
}

#[rocket::get("/boards/<board_id>/summary")]
-fn board_summary(db: State<Db>, board_id: i64) -> Result<Json<BoardSummary>, StdErr> {
+fn board_summary(db: State<Db>, board_id: i64, _t: Token) -> Result<Json<BoardSummary>, StdErr> {
    db.board_summary(board_id).map(Json)
}

#[rocket::delete("/boards/<board_id>")]
- fn delete_board(db: State<Db>, board_id: i64) -> Result<(), StdErr> {
+ fn delete_board(db: State<Db>, board_id: i64, _t: Token) -> Result<(), StdErr> {
    db.delete_board(board_id)
}

// card routes

#[rocket::get("/boards/<board_id>/cards")]
-fn cards(db: State<Db>, board_id: i64) -> Result<Json<Vec<Card>>, StdErr> {
+fn cards(db: State<Db>, board_id: i64, _t: Token) -> Result<Json<Vec<Card>>, StdErr> {
    db.cards(board_id).map(Json)
}

#[rocket::post("/cards", data = "<create_card>")]
-fn create_card(db: State<Db>, create_card: Json<CreateCard>) -> Result<Json<Card>, StdErr> {
+fn create_card(db: State<Db>, create_card: Json<CreateCard>, _t: Token) -> Result<Json<Card>, StdErr> {
    db.create_card(create_card.0).map(Json)
}

#[rocket::patch("/cards/<card_id>", data = "<update_card>")]
fn update_card(
    db: State<Db>,
    card_id: i64,
    update_card: Json<UpdateCard>,
+   _t: Token,
) -> Result<Json<Card>, StdErr> {
    db.update_card(card_id, update_card.0).map(Json)
}

#[rocket::delete("/cards/<card_id>")]
-fn delete_card(db: State<Db>, card_id: i64) -> Result<(), StdErr> {
+fn delete_card(db: State<Db>, card_id: i64, _t: Token) -> Result<(), StdErr> {
    db.delete_card(card_id)
}

pub fn api() -> Vec<rocket::Route> {
    rocket::routes![
        boards,
        create_board,
        board_summary,
        delete_board,
        cards,
        create_card,
        update_card,
        delete_card,
    ]
}
```

Amazing. We did it. The full source code for the Diesel + Rocket implementation can be found in the [companion code repository](https://github.com/pretzelhammer/kanban/tree/main/diesel-rocket) for this article. Okay, now let's do it again, except this time we'll implement it in async Rust using sqlx and actix-web.



## Async Implementation

This section picks up from the same place as the **Sync Implementation** section does: which is right after we added the `serde` & `serde_json` crates to our project's dependencies and created a `models` module.



### SQL Schema Migrations w/sqlx-cli

crates
- sqlx-cli

```bash
cargo install sqlx-cli
```

Again if this fails it's likely because we're missing some development libraries on our system, we can solve this issue with the following commands:

```bash
# macOS
brew install postgresql

# ubuntu
apt-get install pkg-config libssl-dev postgresql libpq-dev
```

And then we can install sqlx-cli with only support for PostgresQL:

```bash
cargo install sqlx-cli --no-default-features --features postgres
```

As before, let's create a boards and cards tables:

```bash
sqlx migrate add create_boards
sqlx migrate add create_cards
```

This creates a `migrations` directory with a couple migration files. Here's the migration file to create the boards table:

```sql
-- create_boards.sql
CREATE TABLE IF NOT EXISTS boards (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT (CURRENT_TIMESTAMP AT TIME ZONE 'utc')
);

-- seed db with some test data for local dev
INSERT INTO boards
(name)
VALUES
('Test board 1'),
('Test board 2'),
('Test board 3');
```

And here's the migration file to create the cards table:

```sql
-- create_cards.sql
CREATE TYPE STATUS AS ENUM ('todo', 'doing', 'done');

CREATE TABLE IF NOT EXISTS cards (
    id BIGSERIAL PRIMARY KEY,
    board_id BIGINT NOT NULL,
    description TEXT NOT NULL,
    status STATUS NOT NULL DEFAULT 'todo',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT (CURRENT_TIMESTAMP AT TIME ZONE 'utc'),
    CONSTRAINT board_fk
        FOREIGN KEY (board_id)
        REFERENCES boards(id)
        ON DELETE CASCADE
);

-- seed db with some test data for local dev
INSERT INTO cards
(board_id, description, status)
VALUES
(1, 'Test card 1', 'todo'),
(1, 'Test card 2', 'doing'),
(1, 'Test card 3', 'done'),
(2, 'Test card 4', 'todo'),
(2, 'Test card 5', 'todo'),
(3, 'Test card 6', 'done'),
(3, 'Test card 7', 'done');
```

We run the migrations with:

```bash
sqlx migrate run
```



### Executing SQL Queries w/sqlx

crates
- sqlx

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
+sqlx = { version = "0.4", features = ["runtime-actix-rustls", "chrono", "postgres"] }
```

Since sqlx is an async library that produces futures those futures have to be executed by a runtime, and actix-web provides a runtime which we're gonna add later, but this is the reason why we added the `runtime-actix-rustls` feature flag.



#### Fetching Data

We can impl the `sqlx::FromRow` trait for our `Board` model using a derive macro:

```diff
// src/models.rs

-#[derive(serde::Serialize)]
+#[derive(serde::Serialize, sqlx::FromRow)]
#[serde(rename_all = "camelCase")]
pub struct Board {
    pub id: i64,
    pub name: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

This impl allows us to use `Board` as the returned result of SQL queries. Let's look at some examples:

```rust
use sqlx::{Connection, PgConnection};
use crate::models::Board;

// example of connecting to PostgresQL
async fn get_connection() -> PgConnection {
    dotenv::dotenv().unwrap();
    let db_url = std::env::var("DATABASE_URL").unwrap();
    PgConnection::connect(&db_url).await.unwrap()
}

// fetch all boards
async fn all_boards(conn: &mut PgConnection) -> Vec<Board> {
    sqlx::query_as("SELECT * FROM boards")
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all boards in order
async fn all_boards_chronological(conn: &mut PgConnection) -> Vec<Board> {
    sqlx::query_as("SELECT * FROM boards ORDER BY created_at ASC")
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch board by primary key
async fn board_by_id(conn: &mut PgConnection, board_id: i64) -> Board {
    sqlx::query_as("SELECT * FROM boards WHERE id = $1")
        .bind(board_id)
        .fetch_one(conn)
        .await
        .unwrap()
}

// fetch all boards with a specific exact name
async fn board_by_name(conn: &mut PgConnection, board_name: &str) -> Vec<Board> {
    sqlx::query_as("SELECT * FROM boards WHERE name = $1")
        .bind(board_name)
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all board whose name contains some string
async fn board_name_contains(conn: &mut PgConnection, contains: &str) -> Vec<Board> {
    // in LIKE queries "%" means "match zero or more of any character"
    let contains = format!("%{}%", contains);
    sqlx::query_as("SELECT * FROM boards WHERE name ILIKE $1")
        .bind(contains)
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all boards created in the past 24 hours
async fn recent_boards(conn: &mut PgConnection) -> Vec<Board> {
    sqlx::query_as("SELECT * FROM boards WHERE created_at >= CURRENT_TIMESTAMP - INTERVAL '1 day'")
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all boards created in the past 24 hours and whose name also contains some string
async fn recent_boards_and_name_contains(conn: &mut PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    sqlx::query_as("SELECT * FROM boards WHERE created_at >= CURRENT_TIMESTAMP - INTERVAL '1 day' AND name ILIKE $1")
        .bind(contains)
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch all boards created in the past 24 hours or whose name contains some string
async fn recent_boards_or_name_contains(conn: &mut PgConnection, contains: &str) -> Vec<Board> {
    let contains = format!("%{}%", contains);
    sqlx::query_as("SELECT * FROM boards WHERE created_at >= CURRENT_TIMESTAMP - INTERVAL '1 day' OR name ILIKE $1")
        .bind(contains)
        .fetch_all(conn)
        .await
        .unwrap()
}
```

Okay, let's look at fetching cards now:

```diff
// src/models.rs

-#[derive(serde::Serialize)]
+#[derive(serde::Serialize, sqlx::FromRow)]
#[serde(rename_all = "camelCase")]
pub struct Card {
    pub id: i64,
    pub board_id: i64,
    pub description: String,
    pub status: Status,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

-#[derive(serde::Deserialize, serde::Serialize)]
+#[derive(serde::Deserialize, serde::Serialize, sqlx::Type)]
#[serde(rename_all = "camelCase")]
+#[sqlx(rename_all = "camelCase")]
pub enum Status {
    Todo,
    Doing,
    Done,
}
```

We can derive `sqlx::Type` for a Rust enum to make it map to a DB enum of the same name. In this case we're mapping the `Status` Rust enum to the `STATUS` DB enum type we defined earlier:

```sql
CREATE TYPE STATUS AS ENUM ('todo', 'doing', 'done');
```

Now let's look at some card query examples:

```rust
use sqlx::{Connection, PgConnection};
use crate::models::{Card, Status};

// fetch all cards
async fn all_cards(conn: &mut PgConnection) -> Vec<Card> {
    sqlx::query_as("SELECT * FROM cards")
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch cards by board
async fn cards_by_board(conn: &mut PgConnection, board_id: i64) -> Vec<Card> {
    sqlx::query_as("SELECT * FROM cards WHERE board_id = $1")
        .bind(board_id)
        .fetch_all(conn)
        .await
        .unwrap()
}

// fetch cards by status
async fn cards_by_status(conn: &mut PgConnection, status: Status) -> Vec<Card> {
    sqlx::query_as("SELECT * FROM cards WHERE status = $1")
        .bind(status)
        .fetch_all(conn)
        .await
        .unwrap()
}
```

Okay, the final select request we need to support is getting the board summary. We first need to add a `From<Vec<(i64, Status)>>` impl for `BoardSummary`:

```rust
// src/models.rs

#[derive(Default, serde::Serialize)]
pub struct BoardSummary {
    pub todo: i64,
    pub doing: i64,
    pub done: i64,
}

// convert list of status counts into a board summary
impl From<Vec<(i64, Status)>> for BoardSummary {
    fn from(counts: Vec<(i64, Status)>) -> BoardSummary {
        let mut summary = BoardSummary::default();
        for (count, status) in counts {
            match status {
                Status::Todo => summary.todo += count,
                Status::Doing => summary.doing += count,
                Status::Done => summary.done += count,
            }
        }
        summary
    }
}
```

And then the function to fetch the board summary is as simple as:

```rust
// fetch board summary
async fn board_summary(conn: &mut PgConnection, board_id: i64) -> BoardSummary {
    sqlx::query_as(
        "SELECT COUNT(*), status FROM cards WHERE board_id = $1 GROUP BY status",
    )
    .bind(board_id)
    .fetch_all(conn)
    .await
    .unwrap()
    .into()
}
```



#### Inserting Data

Examples of creating boards and cards:

```rust
// create board from CreateBoard model
async fn create_board(conn: &mut PgConnection, create_board: CreateBoard) -> Board {
    sqlx::query_as("INSERT INTO boards (name) VALUES ($1) RETURNING *")
        .bind(create_board.name)
        .fetch_one(conn)
        .await
        .unwrap()
}

// create card from CreateCard model
async fn create_card(conn: &mut PgConnection, create_card: CreateCard) -> Card {
    sqlx::query_as("INSERT INTO cards (board_id, description) VALUES ($1, $2) RETURNING *")
        .bind(create_card.board_id)
        .bind(create_card.description)
        .fetch_one(conn)
        .await
        .unwrap()
}
```



#### Updating Data

Example of updating cards:

```rust
// update card from UpdateCard model
async fn update_card(conn: &mut PgConnection, card_id: i64, update_card: UpdateCard) -> Card {
    sqlx::query_as("UPDATE cards SET description = $1, status = $2 WHERE id = $3 RETURNING *")
        .bind(update_card.description)
        .bind(update_card.status)
        .bind(card_id)
        .fetch_one(conn)
        .await
        .unwrap()
}
```



#### Deleting Data

Examples of deleting boards and cards:

```rust
// delete all boards
fn delete_all_boards(conn: &mut PgConnection) {
    sqlx::query("DELETE FROM boards")
        .execute(conn)
        .await
        .unwrap();
}

// delete a board by its id
fn delete_board_by_id(conn: &mut PgConnection, board_id: i64) {
    sqlx::query("DELETE FROM boards WHERE id = $1")
        .bind(board_id)
        .execute(conn)
        .await
        .unwrap();
}

// delete all cards
fn delete_all_cards(conn: &mut PgConnection) {
    sqlx::query("DELETE FROM cards")
    .execute(conn)
    .await
    .unwrap();
}

// delete a card by its id
fn delete_card_by_id(conn: &mut PgConnection, card_id: i64) {
    sqlx::query("DELETE FROM cards WHERE id = $1")
        .bind(card_id)
        .execute(conn)
        .await
        .unwrap();
}

// delete all of the cards on a board
fn delete_cards_by_board(conn: &mut PgConnection, board_id: i64) {
    sqlx::query("DELETE FROM cards WHERE board_id = $1")
        .bind(board_id)
        .execute(conn)
        .await
        .unwrap();
}

// delete all of the done cards on a board
fn delete_done_cards_by_board(conn: &mut PgConnection, board_id: i64) {
    sqlx::query("DELETE FROM cards WHERE board_id = $1 AND status = 'done'")
        .bind(board_id)
        .execute(conn)
        .await
        .unwrap();
}
```

You may have noticed we switched from using `sqlx::query_as` to `sqlx::query`. The difference between the two is that the former attempts to map the result rows into some type which impls `sqlx::FromRow` whereas the latter doesn't, so the latter is more appropriate for `DELETE` queries that don't return any rows.


#### Compile-Time Verification of SQL Queries

One interesting feature of sqlx is that it can verify all of our queries are syntactically and semantically valid at compile-time, but this is behavior we have to opt into by using the `sqlx::query!` and `sqlx::query_as!` macros over the `sqlx::query` and `sqlx::query_as` functions. The downsides of the macros are that they're slightly less ergonomic to use than the regular functions and they will increase compiles but the upsides can greatly outweigh the downsides if we're working in a project with lots of complex queries or many tables and entities. I haven't been using this for this project but they're worth knowing about.



#### Using a Connection Pool w/sqlx

sqlx comes built-in with a connection pool so we don't have to install any additional dependencies!

Here's how we'd create a pool with the default configuration options:

```rust
use sqlx::{Connection, PgConnection, Pool, Postgres};
use sqlx::postgres::PgPoolOptions;

// example of connecting to Pg Pool
async fn get_pool() -> Pool<Postgres> {
    dotenv::dotenv().unwrap();
    let db_url = std::env::var("DATABASE_URL").unwrap();
    PgPoolOptions::new().connect(&db_url).await.unwrap()
}

async fn use_pool(pool: &Pool<Postgres>) {
    // don't need to fetch connection from pool
    // can pass pool directly to queries, e.g.
    sqlx::query_as::<_,(String,)>("SELECT version()")
        .fetch_one(pool) // passing pool directly
        .await
        .unwrap();
}
```




#### Refactoring DB Operations Into a Module

Let's clean up our code and neatly put everything into a single module:

```rust
// src/db.rs

use sqlx::{Connection, PgConnection, Pool, Postgres, postgres::PgPoolOptions};
use crate::models::*;
use crate::StdErr;

pub struct Db {
    pool: Pool<Postgres>,
}

impl Db {
    pub async fn connect() -> Result<Self, StdErr> {
        let db_url = std::env::var("DATABASE_URL")?;
        let pool = PgPoolOptions::new().connect(&db_url).await?;
        Ok(Db { pool })
    }

    pub async fn boards(&self) -> Result<Vec<Board>, StdErr> {
        let boards = sqlx::query_as("SELECT * FROM boards")
            .fetch_all(&self.pool)
            .await?;
        Ok(boards)
    }

    pub async fn board_summary(&self, board_id: i64) -> Result<BoardSummary, StdErr> {
        let counts: Vec<(i64, Status)> = sqlx::query_as(
            "SELECT count(*), status FROM cards WHERE board_id = $1 GROUP BY status",
        )
        .bind(board_id)
        .fetch_all(&self.pool)
        .await?;
        Ok(counts.into())
    }

    pub async fn create_board(&self, create_board: CreateBoard) -> Result<Board, StdErr> {
        let board = sqlx::query_as("INSERT INTO boards (name) VALUES ($1) RETURNING *")
            .bind(&create_board.name)
            .fetch_one(&self.pool)
            .await?;
        Ok(board)
    }

    pub async fn delete_board(&self, board_id: i64) -> Result<(), StdErr> {
        sqlx::query("DELETE FROM boards WHERE id = $1")
            .bind(board_id)
            .execute(&self.pool)
            .await?;
        Ok(())
    }

    pub async fn cards(&self, board_id: i64) -> Result<Vec<Card>, StdErr> {
        let cards = sqlx::query_as("SELECT * FROM cards WHERE board_id = $1")
            .bind(board_id)
            .fetch_all(&self.pool)
            .await?;
        Ok(cards)
    }

    pub async fn create_card(&self, create_card: CreateCard) -> Result<Card, StdErr> {
        let card =
            sqlx::query_as("INSERT INTO cards (board_id, description) VALUES ($1, $2) RETURNING *")
                .bind(&create_card.board_id)
                .bind(&create_card.description)
                .fetch_one(&self.pool)
                .await?;
        Ok(card)
    }

    pub async fn update_card(&self, card_id: i64, update_card: UpdateCard) -> Result<Card, StdErr> {
        let card = sqlx::query_as(
            "UPDATE cards SET description = $1, status = $2 WHERE id = $3 RETURNING *",
        )
        .bind(&update_card.description)
        .bind(&update_card.status)
        .bind(card_id)
        .fetch_one(&self.pool)
        .await?;
        Ok(card)
    }

    pub async fn delete_card(&self, card_id: i64) -> Result<(), StdErr> {
        sqlx::query("DELETE FROM cards WHERE id = $1")
            .bind(card_id)
            .execute(&self.pool)
            .await?;
        Ok(())
    }
}
```

Updated main file:

```diff
// src/main.rs

+mod db;
mod logger;
mod models;

type StdErr = Box<dyn std::error::Error>;

fn main() -> Result<(), StdErr> {
    dotenv::dotenv()?;
    logger::init()?;

    Ok(())
}
```




### HTTP Routing w/actix-web

crates
- actix-web

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
sqlx = { version = "0.4", features = ["runtime-actix-rustls", "chrono", "postgres"] }
+ actix-web = "3.3"
```

Let's throw together a quick actix-web hello world example into our main:

```diff
mod db;
mod logger;
mod models;

type StdErr = Box<dyn std::error::Error>;

+#[actix_web::get("/")]
+async fn hello_world() -> &'static str {
+    "Hello, world!"
+}

+#[actix_web::main]
-fn main() -> Result<(), StdErr> {
+async fn main() -> Result<(), StdErr> {
    dotenv::dotenv().ok();
    logger::init()?;

+   actix_web::HttpServer::new(move || actix_web::App::new().service(hello_world))
+       .bind(("127.0.0.1", 8000))?
+       .run()
+       .await?;

    Ok(())
}
```

We have to make our main function async and decorate it with the `actix_web::main` procedural macro to tell actix-web to start a runtime and execute our main function as the first task.



#### Routing Basics

We can decorate functions with actix-web's procedural macros to route incoming HTTP requests:

```rust
#[actix_web::get("/")]
async fn index() -> &'static str {
    "I'm the index route!"
}

#[actix_web::get("/nested/route")]
async fn nested_route() -> &'static str {
    "I'm a nested route!"
}
```

Data can be extracted from the path using `{param}` placeholders:

```rust
use actix_web::web::Path;

#[actix_web::get("/hello/{name}")]
async fn greet(Path(name): Path<String>) -> String {
    format!("Hello, {}!", name)
}
```

Any type which impls `serde::Deserialize` can be extracted from within `actix_web::web::Path`:

```rust
// actix_web::web::Path can extract anything out of the
// URL path as long as it impls serde::Deserialize
#[actix_web::get("/echo/{string}/{num}/{maybe}/etc")]
async fn echo_path(Path((string, num, maybe)): Path<(String, usize, bool)>) -> String {
    format!("got string {}, num {}, and maybe {}", string, num, maybe)
}

// custom type example
struct EvenNumber(i32);

// hand-written deserialize impl, mostly deferring to i32::deserialize
impl<'de> Deserialize<'de> for EvenNumber {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let value = i32::deserialize(deserializer)?;
        if value % 2 == 0 {
            Ok(EvenNumber(value))
        } else {
            Err(D::Error::custom("not even"))
        }
    }
}

// but now we can extract EvenNumbers directly from the Path:
#[actix_web::get("/even/{even_num}")]
async fn echo_even(Path(even_num): Path<EvenNumber>) -> String {
    format!("got even number {}", even_num.0)
}
```

The response type can be anything that impls `actix_web::Responder`:

```rust
use std::future::{ready, Ready};
use actix_web::{HttpResponse, HttpRequest};
use actix_web::error::InternalError;

// actix_web provides Responder impls for
// - &'static str
// - &'static [u8]
// - String
// - &String
// - some other misc types
// have to use actix_files to get Responder impls for
// - Files
#[actix_web::get("/cargo")]
async fn returns_cargo() -> actix_files::NamedFile {
    actix_files::NamedFile::open("Cargo.toml").unwrap()
}

type ActixError = actix_web::error::Error;

// actix_web also provides Responder impls for
// - Option<T> where T: Responder
//     - returns T if Some(T), 404 Not Found if None
// - Result<T, E> where T: Responder, E: Into<ActixError>
//     - returns T if Ok(T), otherwise ActixError::from(e) if Err(e)

// example custom type
struct EvenNumber(i32);

// example Responder impl
impl Responder for EvenNumber {
    type Error = InternalError<&'static str>;
    type Future = Ready<Result<HttpResponse, Self::Error>>;

    fn respond_to(self, _req: &HttpRequest) -> Self::Future {
        let res = HttpResponse::Ok()
            .set_header("X-Number-Parity", "Even")
            .body(format!("returning even number {}", self.0));
        ready(Ok(res))
    }
}

// returning custom defined type
#[actix_web::get("/even")]
async fn returns_even() -> EvenNumber {
    EvenNumber(2)
}
```



#### GET Requests

We have to pass our `Db` to our actix-web request handlers. We can do this is by passing an instance of the `Db` to the `data` method of the actix-web application factory function, and then we can receive it in our request handlers using the `actix_web::web::Data` extractor.

Since the application factory function creates an application per system thread, the data referenced inside the factory has to be cloneable, so we first have to impl `Clone` for `Db` which is easy since `Pool<Postgres>` already impls `Clone` so we can just derive it:

```diff
// src/db.rs

use sqlx::{Pool, Postgres};

+#[derive(Clone)]
pub struct Db {
    pool: Pool<Postgres>,
}

// etc
```

And then this is how we'd update our main file:

```diff
// src/main.rs

mod db;
mod logger;
mod models;
mod routes;

type StdErr = Box<dyn std::error::Error>;

#[actix_web::get("/")]
async fn hello_world() -> &'static str {
    "Hello, world!"
}

#[actix_web::main]
async fn main() -> Result<(), StdErr> {
    dotenv::dotenv().ok();
    logger::init()?;

+   let db = db::Db::connect().await?;

    actix_web::HttpServer::new(move || {
        actix_web::App::new()
+           .data(db.clone())
            .service(hello_world)
    })
    .bind(("127.0.0.1", 8000))?
    .run()
    .await?;

    Ok(())
}
```

And here's how we'd use the `Data` extractor to receive the `Db` instance in our request handlers:

```rust
use actix_web::web::Data;
use crate::db::Db;

#[actix_web::get("/use/db")]
fn use_db(db: Data<Db>) {
    // use db
}
```

We can automatically return JSON by wrapping the return type in `actix_web::web::Json` which impls `actix_web::Responder` and will serialize the wrapped type to JSON using serde and set the response header `Content-Type: application/json`. Example:

```rust
use actix_web::web::Json;
use crate::models::Status;

#[actix_web::get("/example/json")]
fn return_json() -> Json<Status> {
    Json(Status::Todo)
}
```

Furthermore, because actix-web does not impl `Responder` for `Box<dyn std::error::Error>` we have to wrap it with some type which does, and we can use the generic `actix_web::error::InternalError` type for this purpose:

```rust
use actix_web::error::InternalError;
use actix_web::http::StatusCode;
use crate::StdErr;

fn some_fallible_function() -> Result<&'static str, StdErr> {
    todo!()
}

// map StdErr to an error that impls Responder
fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

#[actix_web::get("/error")]
async fn return_error() -> Result<&'static str, InternalError<StdErr>> {
    some_falliable_function().map_err(to_internal_error)
}
```

Okay, we now have enough context to write all of our GET request handlers:

```rust
use actix_web::web::{Data, Json};
use actix_web::error::InternalError;
use actix_web::http::StatusCode;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, Card, BoardSummary};

// convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

// GET requests

#[actix_web::get("/boards")]
async fn boards(db: Data<Db>) -> Result<Json<Vec<Board>>, InternalError<StdErr>> {
    db.boards()
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::get("/boards/{board_id}/summary")]
async fn board_summary(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<Json<BoardSummary>, InternalError<StdErr>> {
    db.board_summary(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::get("/boards/{board_id}/cards")]
async fn cards(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<Json<Vec<Card>>, InternalError<StdErr>> {
    db.cards(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}
```



#### POST & PATCH Requests

`actix_web::web::Json` not only impls `actix_web::Responder` but also impls `actix_web::FromRequest` which means we can add it as a function parameter to any request handler and actix-web will attempt deserialize the request's JSON body into the type we specify, so here's how we'd write the POST & PATCH request handlers:

```rust
use actix_web::web::{Data, Json};
use actix_web::error::InternalError;
use actix_web::http::StatusCode;

use crate::StdErr;
use crate::db::Db;
use crate::models::{Board, Card, CreateBoard, CreateCard};

// convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

// POST requests

#[actix_web::post("/boards")]
async fn create_board(
    db: Data<Db>,
    create_board: Json<CreateBoard>,
) -> Result<Json<Board>, InternalError<StdErr>> {
    db.create_board(create_board.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/cards")]
async fn create_card(
    db: Data<Db>,
    create_card: Json<CreateCard>,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.create_card(create_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

// PATCH requests

#[actix_web::patch("/cards/{card_id}")]
async fn update_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
    update_card: Json<UpdateCard>,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.update_card(card_id, update_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}
```



#### DELETE Requests

The DELETE request handlers:

```rust
use actix_web::web::{Data, Json};
use actix_web::error::InternalError;
use actix_web::http::StatusCode;

use crate::StdErr;
use crate::db::Db;

// some convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

fn to_ok(_: ()) -> HttpResponse {
    HttpResponse::new(StatusCode::OK)
}

// DELETE requests

#[actix_web::delete("/boards/{board_id}")]
async fn delete_board(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_board(board_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

#[actix_web::delete("/cards/{card_id}")]
async fn delete_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_card(card_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}
```



#### Refactoring API Routes Into a Module

Let's clean up the code and put all of the API routes into their own module:

```rust
// src/routes.rs

use actix_web::web::{Data, Json, Path};
use actix_web::http::StatusCode;
use actix_web::error::InternalError;
use actix_web::dev::HttpServiceFactory;
use actix_web::{HttpResponse, Responder};

use crate::StdErr;
use crate::db::Db;
use crate::models::*;

// some convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

fn to_ok(_: ()) -> HttpResponse {
    HttpResponse::new(StatusCode::OK)
}

// board routes

#[actix_web::get("/boards")]
async fn boards(db: Data<Db>) -> Result<Json<Vec<Board>>, InternalError<StdErr>> {
    db.boards()
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/boards")]
async fn create_board(
    db: Data<Db>,
    create_board: Json<CreateBoard>,
) -> Result<Json<Board>, InternalError<StdErr>> {
    db.create_board(create_board.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::get("/boards/{board_id}/summary")]
async fn board_summary(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<Json<BoardSummary>, InternalError<StdErr>> {
    db.board_summary(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::delete("/boards/{board_id}")]
async fn delete_board(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_board(board_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

// card routes

#[actix_web::get("/boards/{board_id}/cards")]
async fn cards(
    db: Data<Db>,
    Path(board_id): Path<i64>,
) -> Result<Json<Vec<Card>>, InternalError<StdErr>> {
    db.cards(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/cards")]
async fn create_card(
    db: Data<Db>,
    create_card: Json<CreateCard>,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.create_card(create_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::patch("/cards/{card_id}")]
async fn update_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
    update_card: Json<UpdateCard>,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.update_card(card_id, update_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::delete("/cards/{card_id}")]
async fn delete_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_card(card_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

// single public function which returns all of the API request handlers

pub fn api() -> impl HttpServiceFactory + 'static {
    actix_web::web::scope("/api")
        .service(boards)
        .service(board_summary)
        .service(create_board)
        .service(delete_board)
        .service(cards)
        .service(create_card)
        .service(update_card)
        .service(delete_card)
}
```

Updated main file:

```diff
// src/main.rs

mod db;
mod logger;
mod models;
+mod routes;

type StdErr = Box<dyn std::error::Error>;

#[actix_web::get("/")]
async fn hello_world() -> &'static str {
    "Hello, world!"
}

#[actix_web::main]
async fn main() -> Result<(), StdErr> {
    dotenv::dotenv().ok();
    logger::init()?;

    let db = db::Db::connect().await?;

    actix_web::HttpServer::new(move || {
        actix_web::App::new()
            .data(db.clone())
            .service(hello_world)
+           .service(routes::api())
    })
    .bind(("127.0.0.1", 8000))?
    .run()
    .await?;

    Ok(())
}
```



#### Authentication

Let's now implement token-based authentication by checking the `Authorization` HTTP header in requests.

Generating a new migration:

```bash
sqlx migrate add create_tokens
```

The create `tokens` table query:

```sql
-- create_tokens.sql
CREATE TABLE IF NOT EXISTS tokens (
    id TEXT PRIMARY KEY,
    expired_at TIMESTAMP WITH TIME ZONE NOT NULL
);

-- seed db with some local test data
INSERT INTO tokens
(id, expired_at)
VALUES
('LET_ME_IN', (CURRENT_TIMESTAMP + INTERVAL '15 minutes') AT TIME ZONE 'utc');
```

Creating our `Token` model:

```diff
// src/models.rs

+// for authentication
+
+#[derive(sqlx::FromRow)]
+pub struct Token {
+    pub id: String,
+    pub expired_at: chrono::DateTime<chrono::Utc>,
+}

// other models
```

Validating tokens using our `Db`:

```diff
use sqlx::{postgres::PgPoolOptions, Connection, PgConnection, Pool, Postgres};
use crate::models::*;
use crate::StdErr;

#[derive(Clone)]
pub struct Db {
    pool: Pool<Postgres>,
}

impl Db {
    pub async fn connect() -> Result<Self, StdErr> {
        let db_url = std::env::var("DATABASE_URL")?;
        let pool = PgPoolOptions::new().connect(&db_url).await?;
        Ok(Db { pool })
    }

+   // token methods
+
+   pub async fn validate_token<T: AsRef<str>>(&self, token_id: T) -> Result<Token, StdErr> {
+       let token_id = token_id.as_ref();
+       let token = sqlx::query_as("SELECT * FROM tokens WHERE id = $1 AND expired_at > current_timestamp")
+           .bind(token_id)
+           .fetch_one(&self.pool)
+           .await?;
+       Ok(token)
+   }

    // other methods
}
```

There's a couple ways we can implement authentication in actix-web. We can implement it as middleware or as an extractor. Extractors are the actix-web equivalent to Rocket's request guards, and since we implemented authentication using a request guard in Rocket, let's use an extractor for actix-web.

Since everything in actix-web is async, including the `actix_web::FromRequest` method signature, let's add the `futures` crate to our project for some handy utility future traits and functions:

```diff
# Cargo.toml

[package]
name = "kanban"
version = "0.1.0"
edition = "2018"

[dependencies]
dotenv = "0.15"
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
fern = "0.6"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
sqlx = { version = "0.4", features = ["runtime-actix-rustls", "chrono", "postgres"] }
actix-web = "3.3"
+futures = "0.3.14"
```

Okay, so here's the `actix_web::FromRequest` impl for `Token`:

```rust
use std::future::Ready;
use std::pin::Pin;

use actix_web::{FromRequest, HttpRequest};
use actix_web::http::StatusCode;
use actix_web::error::InternalError;
use actix_web::dev::Payload;
use futures::{future, Future, FutureExt};

use crate::StdErr;
use crate::db::Db;
use crate::models::Token;

impl FromRequest for Token {
    type Error = InternalError<&'static str>;
    type Config = ();

    // we return a Future that is either
    // - immediately ready (on a bad request with a missing or malformed Authorization header)
    // - ready later (pending on a SQL query that validates the request's Bearer token)
    type Future = future::Either<
        future::Ready<Result<Self, Self::Error>>,
        Pin<Box<dyn Future<Output = Result<Self, Self::Error>> + 'static>>,
    >;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
        // get request headers
        let headers = req.headers();

        // check that Authorization header exists
        let maybe_auth = headers.get("Authorization");
        if maybe_auth.is_none() {
            return future::err(InternalError::new(
                "missing Authorization header",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // check Authorization header is valid utf-8
        let auth_config = maybe_auth.unwrap().to_str();
        if auth_config.is_err() {
            return future::err(InternalError::new(
                "malformed Authorization header",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // check Authorization header specifies some authorization strategy
        let mut auth_config_parts = auth_config.unwrap().split_ascii_whitespace();
        let maybe_auth_type = auth_config_parts.next();
        if maybe_auth_type.is_none() {
            return future::err(InternalError::new(
                "missing Authorization type",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // check that authorization strategy is using a bearer token
        let auth_type = maybe_auth_type.unwrap();
        if auth_type != "Bearer" {
            return future::err(InternalError::new(
                "unsupported Authorization type",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // check that bearer token is present
        let maybe_token_id = auth_config_parts.next();
        if maybe_token_id.is_none() {
            return future::err(InternalError::new(
                "missing Bearer token",
                StatusCode::BAD_REQUEST,
            ))
            .left_future();
        }

        // we can fetch managed application data using HttpRequest.app_data::<T>()
        let db = req.app_data::<Data<Db>>();
        if db.is_none() {
            return future::err(InternalError::new(
                "internal error",
                StatusCode::INTERNAL_SERVER_ERROR,
            ))
            .left_future();
        }

        // clone these so that we can return an impl Future + 'static
        let db = db.unwrap().clone();
        let token_id = maybe_token_id.unwrap().to_owned();

        async move {
            db.validate_token(token_id)
                .await
                .map_err(|_| InternalError::new("invalid Bearer token", StatusCode::UNAUTHORIZED))
        }
        .boxed_local()
        .right_future()
    }
}
```

Updated routes file:

```diff
use std::pin::Pin;

use actix_web::{FromRequest, HttpRequest, HttpResponse};
use actix_web::web::{Data, Json, Path};
use actix_web::http::StatusCode;
use actix_web::error::InternalError;
use actix_web::dev::{HttpServiceFactory, Payload};
use futures::{Future, FutureExt, future};

use crate::StdErr;
use crate::db::Db;
use crate::models::*;

impl FromRequest for Token {
    type Error = InternalError<&'static str>;
    type Config = ();
    type Future = future::Either<
        future::Ready<Result<Self, Self::Error>>,
        Pin<Box<dyn Future<Output = Result<Self, Self::Error>> + 'static>>,
    >;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
       // impl
    }
}

// some convenience functions

fn to_internal_error(e: StdErr) -> InternalError<StdErr> {
    InternalError::new(e, StatusCode::INTERNAL_SERVER_ERROR)
}

fn to_ok(_: ()) -> HttpResponse {
    HttpResponse::new(StatusCode::OK)
}

// board routes

#[actix_web::get("/boards")]
async fn boards(
    db: Data<Db>,
+   _t: Token
) -> Result<Json<Vec<Board>>, InternalError<StdErr>> {
    db.boards()
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/boards")]
async fn create_board(
    db: Data<Db>,
    create_board: Json<CreateBoard>,
+   _t: Token,
) -> Result<Json<Board>, InternalError<StdErr>> {
    db.create_board(create_board.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::get("/boards/{board_id}/summary")]
async fn board_summary(
    db: Data<Db>,
    Path(board_id): Path<i64>,
+   _t: Token,
) -> Result<Json<BoardSummary>, InternalError<StdErr>> {
    db.board_summary(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::delete("/boards/{board_id}")]
async fn delete_board(
    db: Data<Db>,
    Path(board_id): Path<i64>,
+   _t: Token,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_board(board_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

// card routes

#[actix_web::get("/boards/{board_id}/cards")]
async fn cards(
    db: Data<Db>,
    Path(board_id): Path<i64>,
+   _t: Token,
) -> Result<Json<Vec<Card>>, InternalError<StdErr>> {
    db.cards(board_id)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::post("/cards")]
async fn create_card(
    db: Data<Db>,
    create_card: Json<CreateCard>,
+   _t: Token,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.create_card(create_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::patch("/cards/{card_id}")]
async fn update_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
    update_card: Json<UpdateCard>,
+   _t: Token,
) -> Result<Json<Card>, InternalError<StdErr>> {
    db.update_card(card_id, update_card.0)
        .await
        .map(Json)
        .map_err(to_internal_error)
}

#[actix_web::delete("/cards/{card_id}")]
async fn delete_card(
    db: Data<Db>,
    Path(card_id): Path<i64>,
+   _t: Token,
) -> Result<HttpResponse, InternalError<StdErr>> {
    db.delete_card(card_id)
        .await
        .map(to_ok)
        .map_err(to_internal_error)
}

pub fn api() -> impl HttpServiceFactory + 'static {
    actix_web::web::scope("/api")
        .service(boards)
        .service(board_summary)
        .service(create_board)
        .service(delete_board)
        .service(cards)
        .service(create_card)
        .service(update_card)
        .service(delete_card)
}
```

We did it! Again! Look at us go, we're implementation machines! The full source code for the sqlx + actix-web implementation can be found in the [companion code repository](https://github.com/pretzelhammer/kanban/tree/main/sqlx-actix-web) for this article.

Since we have two identical RESTful API servers but with two different implementations let's benchmark them and see which is faster :)



## Benchmarks



### Servers

I don't normally benchmark RESTful API servers so I don't know what "good performance" is and what "bad performance" is. Aside from comparing the Diesel + Rocket server to the sqlx + actix-web server I've decided to throw in a couple node.js servers to make things more interesting. The full source code for the benchmarks as well as all the servers can be found in the [companion code repository](https://github.com/pretzelhammer/kanban) for this article. Here's a list of the servers we will be benchmarking:

Server #1: Diesel + Rocket
- Nickname: DR
- Connection pool: r2d2
- SQL executor: Diesel
- HTTP routing: Rocket
- Compiled with: Rust v1.53 (Nightly)

Server #2: sqlx + actix-web
- Nickname: SA
- Connection pool: sqlx
- SQL executor: sqlx
- HTTP Routing: actix-web
- Compiled with: Rust v1.53 (Nightly)

Server #3: pg-promise + express.js (single process)
- Nickname: PES
- Connection pool: pg-promise
- SQL executor: pg-promise
- HTTP Routing: express.js
- Interpreted with: node.js v16.0.0
- Mode: single process

Server #4: pg-promise + express.js (multi process)
- Nickname: PEM
- Connection pool: pg-promise
- SQL executor: pg-promise
- HTTP Routing: express.js
- Interpreted with: node.js v16.0.0
- Mode: multi process 

Test Machine: MacBook Pro (2019)
- CPU: 2.3 Ghz 8-core Intel Core i9
- Memory: 32 GB 2667 MHz DDR4



### Methodology

The tools we're going to use for benchmarking and profiling are vegeta and psutil. Vegeta is an HTTP load testing command line tool written in Go. We can give it a list of targets, a duration, and a number of workers and it will pummel the targets for the given duration using the given number of workers while recording statistics like the number of requests successfully processed, their response times, their status codes, and so on. Psutil is a Python library that we can use to easily write a script that queries the system every second to check how much CPU and memory a process is using.

Let's run all of the HTTP load tests for 60 seconds and use up to a max of 40 workers. Vegeta will naturally scale the workers if the HTTP server can handle the load. Also, let's use two different sets of targets, the first set is going to represent a read-only (RO) workload and the second set is going to represent a more realistic reads + writes (RW) workload.

Here's the RO workload target list:

```none
GET http://localhost:8000/api/boards
Authorization: Bearer LET_ME_IN

GET http://localhost:8000/api/boards/1/summary
Authorization: Bearer LET_ME_IN

GET http://localhost:8000/api/boards/1/cards
Authorization: Bearer LET_ME_IN
```

Here's the RW workload target list:

```none
GET http://localhost:8000/api/boards
Authorization: Bearer LET_ME_IN

GET http://localhost:8000/api/boards/1/summary
Authorization: Bearer LET_ME_IN

POST http://localhost:8000/api/boards
Authorization: Bearer LET_ME_IN
Content-Type: application/json
@post-board.json

DELETE http://localhost:8000/api/boards/10000
Authorization: Bearer LET_ME_IN

GET http://localhost:8000/api/boards/1/cards
Authorization: Bearer LET_ME_IN

POST http://localhost:8000/api/cards
Authorization: Bearer LET_ME_IN
Content-Type: application/json
@post-card.json

PATCH http://localhost:8000/api/cards/1
Authorization: Bearer LET_ME_IN
Content-Type: application/json
@patch-card.json

DELETE http://localhost:8000/api/cards/10000
Authorization: Bearer LET_ME_IN
```

And here are the test JSON payloads:

```js
// post-board.json
{"name": "Vegeta Stress Test Board"}

// post-card.json
{"boardId": 3, "description": "Vegeta Stress Test Card"}

// patch-card.json
{"description": "Vegeta Stress Update Card", "status": "doing"}
```

Of course we're going to compile the Rust servers with this command:

```bash
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

And with this release profile:

```toml
[profile.release]
debug = 0
lto = true
codegen-units = 1
panic = "abort"
```

Also, let's disable the logging in the Rust servers for the benchmarks because I didn't bother adding any logging to the node.js servers.



### Calculating Best Possible Performance

Since all of our servers are connecting to the same PostgresQL database it's possible that PostgresQL might become the common bottleneck between them all and our benchmarks essentially become useless. For starters, let's run PostgresQL entirely in memory. Second, let's benchmark PostgresQL itself so we can determine the theoretical upper bound for the performance of our RESTful API servers.

So benchmarking PostgresQL is really easy! If we have PostgresQL installed on a machine it also comes with a command line tool called `pgbench` that does all the hard work for us. Initializing the benchmark tables is as easy as:

```bash
pgbench --initialize
```

Running a read-only workload is as easy as:

```bash
pgbench --username postgres --client 8 --jobs 8 --transactions 1000 --select-only
```

I chose 8 as my number of clients and jobs because it's equal to the number of physical cores on my test machine. Here are the results I got:

```none
starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 8
number of transactions per client: 1000
number of transactions actually processed: 8000/8000
latency average = 0.244 ms
tps = 32796.428469 (including connections establishing)
tps = 33791.866076 (excluding connections establishing)
```

Wow! I know a big number when I see one and that number is HUGE! Apparently I can get ~34k read-only transactions per second (tps) on my test machine. I'm using the number which excludes establishing connections because all of the servers use connection pools and re-use connections between requests.

Let's run a more realistic reads + writes workload:

```bash
pgbench --username postgres --client 8 --jobs 8 --transactions 1000
```

The results:

```none
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 8
number of transactions per client: 1000
number of transactions actually processed: 8000/8000
latency average = 2.530 ms
tps = 3162.045587 (including connections establishing)
tps = 3168.014678 (excluding connections establishing)
```

That number is not quite as big or mind-blowing as the previous number, but almost ~3.2k transactions per second (tps) for a workload of reads and writes still seems pretty good.

Okay, so we know that every API request to our servers performs exactly two SQL queries: the first to validate the bearer token in the authorization header and the second to select, insert, update, or delete some data. The math's really simple, but here's a table anyway:

| Workload | PostgresQL Performance | Best Possible RESTful API Server Performance |
|-|-|-|
| **Read-Only** | ~34k transactions per second | ~17k requests per second |
| **Reads + Writes** | ~3.2k transactions per second | ~1.6k requests per second |



### Measuring Resource Usage

I'm going to measure CPU usage in CPU seconds and memory usage in megabytes. Everyone knows what megabytes are so I'm not going to explain those, but not everyone is familiar with CPU seconds and they're kinda weird so let's discuss those now.

When people say "CPU second" what they really mean is "CPU logical core second." For example, a CPU with 16 logical cores can perform 16 CPU seconds of processing for every second of wall clock time. This is why when you open the process manager tool in your operating system of choice you'll occasionally see it report some processes as using over 100% CPU. This makes no sense, as it's not possible to use more than 100% of anything, but it's reported this way because the percent is calculated based on the processing power of a single logical core of the CPU and not the total processing power of the entire CPU. For example, a process running on a CPU with 16 logical cores can use up to "1600%" of the CPU. Yup, it's dumb, but whatever, now you know.



### Results

As a quick reminder, the servers we're benchmarking:
- DR: Diesel + Rocket
- SA: sqlx + actix-web
- PES: pg-promise + express.js (single process)
- PEM: pg-promise + express.js (multi process)

The workloads we're using for the benchmarks:
- Read-only (RO) workload for 60 seconds with up to 40 workers
- Reads + Writes (RW) workload for 60 seconds with up to 40 workers

And the stats we're tracking:
- Total requests processed
- How many of the requests were successful (return 200 status code)
- CPU usage (in CPU seconds)
- Memory usage (in megabytes)

And the theoretical upper bounds we calculated:
- Server should be able to process up to ~17k requests per second for RO workload (before DB becomes bottleneck)
- Server should be able to process up to ~1.6k requests per second for RW workload (before DB becomes bottleneck)
- Server can use up to 16 CPU seconds per second (since the test machine has 16 logical cores)
- Server can use up to 32 GB of memory (since that's how much the test machine has)



#### Read-Only Workload

**Request Throughput**

| Server | Total Requests | Successful Requests | Success Rate | Successful Requests per Second |
|-|-|-|-|-|
| **DR** | 23571 req | 23562 req | 99.96% | 392 req/sec |
| **SA** | 170714 req | 170714 req | 100% | 2845 req/sec |
| **PES** | 148737 req | 148737 req | 100% | 2479 req/sec |
| **PEM** | 243474 req | 243474 req | 100% | 4058 req/sec |

**Request Latencies**

| Server | Min | Avg | 50th percentile | 90th percentile | 95th percentile | 99th percentile | Max |
|-|-|-|-|-|-|-|-|
| **DR** | 6.8 ms | 102.0 ms | 26.5 ms | 316.4 ms | 327.7 ms | 513.7 ms | 1.78 s |
| **SA** | 6.2 ms | 14.0 ms | 12.8 ms | 17.9 ms | 20.4 ms | 27.4 ms | 1.03 s |
| **PES** | 5.8 ms | 16.1 ms | 15.8 ms | 20.2 ms | 21.7 ms | 26.3 ms | 1.02 s |
| **PEM** | 3.5 ms | 9.8 ms | 8.6 ms | 14.6 ms | 17.7 ms | 25.7 ms | 1.01 s |

**Resource Usage**

| Server | Total CPU Seconds | Avg CPU Seconds per Second | Avg CPU Seconds per Successful Request | Max Memory Used |
|-|-|-|-|-|
| **DR** | 8.500 cpu | 0.142 cpu/sec | 0.00036 cpu/req | 17.32 MB |
| **SA** | 73.806 cpu | 1.230 cpu/sec | 0.00043 cpu/req | 15.75 MB |
| **PES** | 60.202 cpu | 1.003 cpu/sec | 0.00040 cpu/req | 110.75 MB |
| **PEM** | 184.945 cpu | 3.082 cpu/sec | 0.00076 cpu/req | 806.94 MB |

None of the servers even came close to hitting the theoretical ~17k requests per second maximum but I honestly wasn't expecting them to so I'm not really disappointed by that.

The DR server totally bombed. I have no clue how it's _that slow_. Yeah, it barely used any CPU and it barely used any memory but that's also because it barely processed any requests. It's also the only server that failed to process all requests! I asked for performance debugging help in both the Diesel and Rocket repos the conclusion was that the cause of the poor performance is Rocket and not Diesel. Also, there's no way to fix it apparently, Rocket v0.4 is just really slow.

The SA server did really well in my opinion! A throughput of 2845 req/sec with an average latency of 14 ms per request while using only 15.75 MB of memory is pretty amazing.

The PES server was surprisingly competitive. It almost had the same performance as the SA server in terms of request throughput and average latency, however the SA server is multi-threaded and is using all 16 of my test machine's CPU cores while the PES server is single-threaded and only using a single core. With that mind, the PES server's performance is a little bit more impressive than the SA server's performance in my opinion. It uses way more memory of course, but that's to be expected since it's node.js.

Wow, the PEM server totally stole the show here. All of its numbers are impressive, including the bad ones. Compared to the other servers, a 4058 req/sec throughput with an average 10 ms latency is bonkers. However, it also overwhelmingly used by far the most CPU with an average of 3.082 cpu/sec and by far the most memory at 806.94 MB.



#### Reads + Writes Workload

**Request Throughput**

| Server | Total Requests | Successful Requests | Success Rate | Successful Requests per Second |
|-|-|-|-|-|
| **DR** | 26365 req | 21063 req | 79.89% | 351 req/sec |
| **SA** | 71522 req | 71522 req | 100% | 1192 req/sec |
| **PES** | 37767 req | 37767 req | 100% | 629 req/sec |
| **PEM** | 68968 req | 68968 req | 100% | 1149 req/sec |

**Request Latencies**

| Server | Min | Avg | 50th percentile | 90th percentile | 95th percentile | 99th percentile | Max |
|-|-|-|-|-|-|-|-|
| **DR** | 0.1 ms | 91.1 ms | 22.2 ms | 317.4 ms | 329.3 ms | 356.7 ms | 13.62 s |
| **SA** | 6.7 ms | 33.6 ms | 21.5 ms | 74.7 ms | 96.8 ms | 138.2 ms | 1.17 s |
| **PES** | 7.6 ms | 63.6 ms | 49.2 ms | 89.2 ms | 198.6 ms | 338.1 ms | 411.1 ms |
| **PEM** | 2.8 ms | 34.8 ms | 17.0 ms | 79.7 ms | 115.2 ms | 206.0 ms | 1.15 s |

**Resource Usage**

| Server | Total CPU Seconds | Avg CPU Seconds per Second | Avg CPU Seconds per Successful Request | Max Memory Used |
|-|-|-|-|-|
| **DR** | 13.024 cpu | 0.217 cpu/sec | 0.00062 cpu/req | 34.25 MB |
| **SA** | 365.514 cpu | 6.092 cpu/sec | 0.00511 cpu/req | 125.40 MB |
| **PES** | 66.691 cpu | 1.112 cpu/sec | 0.00177 cpu/req | 167.29 MB |
| **PEM** | 291.138 cpu | 4.852 cpu/sec | 0.00422 cpu/req | 1149.82 MB |

I'm surprised by these results! I was expecting the results from the RW workload to look similar to the results from the RO workload but be just slightly worse. However, having to do writes must have increased the contention for locks within the DB, which in turn has increased the average latency to handle every request, and the larger number of longer-lived requests seem to have totally tanked the performance of the node.js servers. As before, let's discuss the results server by server.

Again, the DR server totally bombed. It's performance looks slightly less worse in this context than in the previous context but it's still terrible. Also again, Rocket is the cause of the poor performance here, not Diesel.

The SA server had a 1192 req/sec throughput with an average latency of 33.6 ms and a maximum memory usage of 125.40 MB. It also ate up a ton of CPU, at an average of 6.092 cpu/sec.

The PES server's performance really tanked, it's no longer competitive with the SA server. It seems like node.js really shines when it can handle lots of small requests quickly. If the complexity of the requests and their time to complete increases it seems like node.js starts to really struggle.

Despite it's awesome performance in the RO workload the PEM server's performance also greatly fell in the RW workload, but despite that it was still competitive with the SA server! Not only is it competitive in terms of request throughput and average request latency but PEM also used less CPU! However, it used tremendously more memory, at 1149.82 MB.

After seeing all of the results of both workloads it's hard to crown an overall winner.

If we were working in an environment where we only had a single core CPU to work with, or CPU time was really expensive and we wanted to use CPU as efficiently as possible, then the overall winner hands down is PES.

If we were working in a very memory constrained environment, or memory was really expensive to use, then the overall winner hands down is SA.

But if we didn't particularly care about CPU or memory usage, and just wanted the highest request throughput we could get from a single machine with a multi-core CPU and plenty of memory, then the overall winner is PEM, although SA would be a reasonable runner-up choice.



## Concluding Thoughts



### Diesel vs sqlx

I'm gonna be upfront about my biases here: I love SQL and I hate ORMs.

In my opinion, SQL is already an amazing abstraction. It's high-level, elegant, declarative, flexible, composable, intuitive, and powerful. ORMs which attempt to abstract over SQL rarely capture all of these qualities, and usually the result is an underpowered, inflexible, leaky API that gives users the ability to perform only a small fraction of the queries that they could more easily and concisely express in SQL.

What I like about Diesel v1.4:
- diesel-cli is really nice, especially for authoring, running, and reverting migrations.
- The derive macros `diesel::Queryable`, `diesel::QueryableByName`, `diesel::Insertable`, and `diesel::AsChangeSet` are pretty nice.

Where I think Diesel v1.4 could improve:
- More guides would be nice, I think it's a bit strange that associations are not covered in any guide and the only way to learn about that feature is to stumble across it in the API docs.
- More logging would be nice, if I set the log level to `TRACE` I see nothing from Diesel.
- Something like a `diesel_contrib` crate, similar to how Rocket has `rocket` for core stuff and `rocket_contrib` for commonly requested nice-to-haves, would be great. It was not fun having to find and use an unofficial 3rd-party crate just to map DB enums to Rust enums.
- Almost all popular Rust libraries use macros, but Diesel especially seems to use a ton. It's still unclear to me why there needs to be a `diesel::Queryable` and a `diesel::QueryableByName` macro, as those seem like they can be consolidated. Also, if the generated Diesel schema file is not suppose to be edited by hand, why does it use macros at all? Why not just generate the code the macros would generate in the first place?
- Diesel overall felt underwhelming compared to feature-rich and batteries-included ORMs available in other languages. I think calling Diesel an "ORM" probably falsely expectations for a lot of users, or at least it did for me. I've seen similar libraries in other languages call themselves "micro ORMs" or "query builders" which I think would be much more appropriate descriptions for Diesel.
- No support for async/await and because the implementation seems to be blocked by some Rust compiler bug that nobody is interested in fixing it seems like async/await is not going to come to Diesel any time soon.

What I like about sqlx v0.4:
- Lets me just write and run SQL, which is what I want to do in the first place anyway.
- The derive macros `sqlx::FromRow` and `sqlx::Type` are really nice.
- Logs every executed query, how many rows it returned, and how long the query took at an `INFO` level. Very nice!

Where I think sqlx v0.4 can improve:
- More documentation please! The docs on sqlx-cli especially are almost nonexistent.
- Please add all of diesel-cli's migration-related functionality to sqlx-cli, including the ability to revert migrations.
- Derive macros similar to `diesel::Insertable` and `diesel::AsChangeSet` which I could use to decorate structs and then pass those structs as-is to the `bind` method of parameterized queries would be pretty nice.

The benchmarks we performed above were skewed a lot by Rocket and actix-web, so it's not really fair to use them to compare Diesel and sqlx. If you would like to see the results of detailed benchmarks run specifically to profile Diesel and sqlx then you can find [the results for those here](https://github.com/diesel-rs/metrics/) and [the source code for them here](https://github.com/diesel-rs/diesel/tree/master/diesel_bench). Disclaimer: this benchmark suite is maintained by the Diesel team.



### Rocket vs actix-web

What I like about Rocket v0.4:
- Super good DX (developer experience)!
- Amazing documentation!
- Amazing logs!
- Procedural macros check that all path and data parameters are used in the request handler function!
- All of the guard-related traits are great: `FromRequest`, `FromParam`, `FromData`, and so on.

Where I think Rocket v0.4 can improve:
- Please get off nightly Rust and use stable Rust. (Note: this is coming in Rocket v0.5)
- Please support async/await. (Note: this is coming in Rocket v0.5)
- Please improve performance!

What I like about actix-web v3.3:
- Good documentation.
- Using procedural macros to decorate request handlers is optional and there's a non-macro API.
- Nice performance.

Where I think actix-web v3.3 can improve:
- Providing a `Responder` impl for `()` that returns `204 No Content` would be really nice.
- Providing a `ResponseError` impl for `E where E: std::error::Error` that returns `500 Server Error` and prints the error message on debug builds and prints a generic error message on release builds would be really nice.
- Please log more, like way more! Even when I set the logging level to `TRACE` the logs were still almost useless when it came to helping me debug issues.
- On one hand, the `FromRequest` impl on `Path<T> where T: DeserializeOwned` is lowkey brilliant, but on the other hand if the type is not trivially deserializable (e.g. any situation where a member has to be validated) then it's hostile to users who have never written a `serde::Deserialize` impl by hand before, which is most users.
- Nice performance, but could be nicer :)



### Sync Rust vs Async Rust

We didn't really get into "advanced" async programming in this article, and in a way that's a good thing! One of the big selling points of introducing the `async` and `await` keywords to Rust is that it would make async programming as simple and straight-forward as sync programming, and I believe they delivered on that promise in this project given how similar the async implementation was to the sync implementation.

Although with that said, while async Rust _can be_ as simple as sync Rust, it also can be way more complicated! I wasn't completely forthcoming or transparent about all of my struggles in this article, but writing the `actix_web::FromRequest` impl for `Token` was really, really hard. There were also some things I tried to do in the async implementation that never made it into the article because I just couldn't get them to compile. While I believe this is _partially_ due to my own inexperience with async programming Rust, I also think the async stuff is just inherently harder and more complex. I'd like to tackle all of these things head-on so I'll probably write an _"Advanced Async Patterns in Rust"_ article or something like that in the future.


### Rust vs JS

I noticed I _feel_ very productive in Rust because I'm making a lot of decisions, but most of the time most of the decisions are unrelated to solving the problem at hand, so my _actual_ progress and productivity is kinda low. I'm well past the newbie stage so I rarely have issues with lifetimes or borrowing anymore, but like I mentioned above: working with futures and async Rust can still be really challenging.

The one big thing I missed from Rust when re-writing the RESTful API servers in JS was the static type checking, but that's not really an argument for Rust, it's more of an argument for TypeScript. One of my personal takeaways from this project is that I should probably learn TypeScript. 



### In Summary

In the future, if I find myself writing another RESTful API server in Rust, I'm definitely going to use sqlx over Diesel. If I had to pick an web framework right now I'd pick actix-web over Rocket because the performance difference is too great to ignore but I will say Rocket is absolutely better than actix-web in almost every other possible metric: documentation, logging, easy-to-use APIs, and overall developer friendliness. I'm eagerly awaiting the release of Rocket v0.5, which should hopefully fix most of the issues I had with Rocket v0.4, including improving the overall performance of the framework.



## Discuss

Discuss this article on
- [Github](https://github.com/pretzelhammer/rust-blog/discussions)
- [learnrust subreddit](https://www.reddit.com/r/learnrust/comments/nanar9/restful_api_in_sync_async_rust/)
- [official Rust users forum](https://users.rust-lang.org/t/blog-post-restful-api-in-sync-async-rust/59713)
- [Twitter](https://twitter.com/pretzelhammer/status/1392825891428909062)



## Notifications

Get notified when the next article get published by
- [Following pretzelhammer on Twitter](https://twitter.com/pretzelhammer) or
- [Subscribing to this repo's release RSS feed](https://github.com/pretzelhammer/rust-blog/releases.atom) or
- Watching this repo's releases (click `Watch` -> click `Custom` -> select `Releases` -> click `Apply`)



## Further Reading

- [Common Rust Lifetime Misconceptions](./common-rust-lifetime-misconceptions.md)
- [Sizedness in Rust](./sizedness-in-rust.md)
- [Tour of Rust's Standard Library Traits](./tour-of-rusts-standard-library-traits.md)
- [Learning Rust in 2020](./learning-rust-in-2020.md)
- [Learn Assembly with Entirely Too Many Brainfuck Compilers](./too-many-brainfuck-compilers.md)
