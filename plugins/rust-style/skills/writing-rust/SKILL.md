---
name: writing-rust
description: Use when writing or reviewing Rust — designing domain types, modeling state with enums/sum types, choosing dependency-injection boundaries, building structs with many fields, parsing or converting values, handling errors, or writing async code. Style is types-first and closures-as-ports, keeping business logic pure and unit-testable with no DB, mocks, or external services.
---

# Writing Rust

## Overview

Two load-bearing ideas drive this style:

1. **Push logic into types.** If the compiler can reject an illegal state, a test never has to. Model domains with sum types and newtypes so wrong values can't be *built*, let alone stored.
2. **Inject behavior as closures, not traits.** Business logic takes its side effects (DB reads, clock, provisioning) as `async` closure parameters. The logic layer stays pure and is unit-tested by passing in-memory closures — no DB, no mocks, no trait objects.

Everything below serves those two ideas. Business rules live in a pure `logic`/`domain` layer; I/O is wired in at the edge (the service/handler); transport concerns (HTTP status, JSON) are translated last.

## When to use

Use for any Rust design decision: a new domain type, a function that needs a DB, a struct with many fields, a value coming in as a string, a new error case. When NOT to force it: tiny scripts, or a genuine plugin system with many swappable implementations discovered at runtime (there, a trait object earns its place — see the decision guide).

## The principles (quick reference)

| Situation | Do this | Not this |
|---|---|---|
| A value is one of N shapes | `enum` with per-variant data | struct with nullable fields + a `kind` tag |
| A primitive has meaning (`id`, `email`) | newtype `struct Id(u32)` | bare `u32` / `String` everywhere |
| A string/number arrives from outside | `parse() -> Result<T, E>` constructor | validate-in-place, pass primitive on |
| Logic needs a DB / clock / external call | take it as an `async` closure param | define a repository `trait` |
| Struct with many/optional/defaulted fields | `#[derive(TypedBuilder)]` | positional `new(...)` with 8 args |
| String ⇄ enum conversion | `strum::Display` + `EnumString` | hand-written match arms |
| Domain error | `thiserror` enum in the domain layer | `String` errors / panic / `anyhow` in core |
| Abstraction at a hot boundary | generics / `impl Trait` (monomorphized) | `Box<dyn>` unless you must erase the type |
| Two+ awaits that don't depend on each other | `tokio::try_join!` / `join!` / `try_join_all` | `.await` them one after another |

## Make illegal states unrepresentable

Model each variant's data on the variant. A root node has no id; a child has a level and an id. Don't represent that as `struct { is_root: bool, level: Option<Level>, id: Option<u32> }` — that admits "root with an id" and "child with no level."

```rust
pub enum NodeId {
    Root,
    Node { level: Level, id: u32 },
}
```

Closed sets are enums, not strings. Modeling `purpose` as an enum makes "a sensor measures a *known* resource" a type-level invariant — an unknown value can't be built, stored, or read back. Derive string conversion instead of writing it:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq,
         strum::Display, strum::EnumString, strum::EnumIter)]
#[strum(ascii_case_insensitive)]
pub enum Resource {
    #[strum(serialize = "electricity")]      Electricity,
    #[strum(serialize = "district_heating")] DistrictHeating,
    Gas, Water,
}
// "electricity".parse::<Resource>()  and  resource.to_string()  are free.
// Resource::iter() enumerates all variants (great for tests / OpenAPI schemas).
```

Let a variant carry only the data it needs — here only one variant holds a payload:

```rust
pub enum EdgeKind {
    HasLabel(String),   // carries data
    HasSensor,          // doesn't
    Blocked,
}
```

## Newtypes over primitives + parse, don't validate

Wrap meaningful primitives so the type system tracks intent, and make the *only* way to build one a constructor that enforces the invariant. A caller holding an `Email` knows it's well-formed — the check happened once, at the boundary.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Email(String);           // private field: no bypassing the constructor

impl Email {
    /// Parse, don't validate: returns a *valid* value or an error, never a
    /// "maybe-valid" primitive that later code must re-check.
    pub fn parse(s: &str) -> Result<Email, String> {
        match s.split_once('@') {
            Some((u, d)) if !u.is_empty() && d.contains('.') => Ok(Email(s.to_owned())),
            _ => Err(format!("not an email: {s}")),
        }
    }
}
```

Prefer inherent `parse(&str) -> Result<Self, E>` for domain newtypes. Reach for `TryFrom` only when a generic bound or ecosystem trait actually requires it.

## Closures as ports — the signature idiom

**Business logic functions take their effects as generic `async` closures.** No repository trait. The logic is a pure function of its inputs plus the operations you hand it, so a test calls it with in-memory closures and prod calls it with DB-backed ones.

```rust
// logic layer — pure, no DB import, no trait
pub async fn get_node<F, Fut>(id: NodeId, get_node_fn: F) -> Result<Node, RepoError>
where
    F: FnOnce(NodeId) -> Fut,
    Fut: Future<Output = Result<Option<Node>, RepoError>>,
{
    get_node_fn(id.clone()).await?.ok_or(RepoError::NotFound(id))
}
```

Choose the closure trait by call count: `FnOnce` if the op is called once, `Fn` if the logic may call it multiple times (say it in a comment — future-you will wonder). Add `+ Send` when the future crosses a Tokio boundary.

**Prod wires real closures via factory functions** returning `impl Fn(..) -> Future`. Box the future behind a type alias to keep signatures readable:

```rust
pub type RepoFut<T> = Pin<Box<dyn Future<Output = Result<T, RepoError>> + Send>>;

// `+ Clone` lets one factory output satisfy both Fn and FnOnce call sites.
pub fn get_node_fn(db: &'static Db, table: String)
    -> impl Fn(NodeId) -> RepoFut<Option<Node>> + Clone
{
    move |id| {
        let t = table.clone();
        Box::pin(async move { db_get_node(db, &t, &id).await })
    }
}
```

**Tests inject the store directly** — this is the whole payoff, no mock framework:

```rust
let store = Store::new();
let node = get_node(id, |x| async move { Ok(store.get(&x)) }).await?;
```

Synchronous strategy seams work the same way with `&dyn Fn`: a formula evaluator takes `resolve: &dyn Fn(&str) -> f64` instead of a `Resolver` trait; a testable clock is a stored `Box<dyn Fn() -> DateTime<Utc>>` injected via `with_clock(...)`.

### Closures vs. a trait — the decision

Default to closures. A trait earns its place only when:

- the port has **several methods that share state/lifecycle** (a closure-per-method gets unwieldy past ~3), **or**
- you need a **collection of interchangeable implementations** selected at runtime (`Vec<Box<dyn Port>>`), **or**
- the port is a **published extension point** others implement.

One or two operations, injected for testability → closure. Many cohesive operations or open-ended implementors → trait.

## Independent awaits run concurrently

**Required:** if one future doesn't need another's result, await them together. Sequential `.await`s on independent I/O add their latencies for nothing. Before writing a second `.await`, ask: does this call use the previous result? If not, join.

```rust
// Wrong: the second fetch waits for the first for no reason
let node = get_node_fn(id).await?;
let sensors = list_sensors_fn(site).await?;

// Right: both in flight at once; first error short-circuits
let (node, sensors) = tokio::try_join!(get_node_fn(id), list_sensors_fn(site))?;
```

- `try_join!` for `Result` futures, `join!` when you need every outcome.
- N futures over a collection → `futures::future::try_join_all(ids.into_iter().map(&get_node_fn))`, not a `for` loop with `.await` inside. Unbounded fan-out against a rate-limited service → `stream::iter(..).buffer_unordered(n)`.
- Calling one closure port in several concurrent futures needs `Fn`, not `FnOnce`.
- Stay sequential only when there is a real dependency: data (B needs A's output), ordering (write-then-read), or a short-circuit (skip B when A fails or returns `None`). Say which in a comment.

## Typed builder for many-field structs

Use the `typed-builder` crate — the derive enforces at compile time that every required field is set before `.build()`, and defaults can reference earlier fields.

```rust
#[derive(Clone, Debug, PartialEq, TypedBuilder)]
pub struct User {
    pub email: String,
    #[builder(default = Email::parse(&email).unwrap().into_id())] // default from another field
    pub id: UserId,
    pub name: String,
    #[builder(default)]                    pub language: Language,   // uses Default
    #[builder(default = Utc::now())]       pub created: DateTime<Utc>,
}
// User::builder().email(e).name(n).build()   — omitting `name` won't compile.
```

This replaces both the 8-argument `new` and the hand-rolled type-state builder — reach for a manual phantom-typed builder only if you need states the derive can't express.

## Errors: model in the domain, translate at the edge

Domain errors are a `thiserror` enum that can embed domain types. The core never returns `String` errors and never panics on expected failure. `anyhow` is fine at the *outer* adapter boundary; keep it out of the domain.

```rust
#[derive(Debug, thiserror::Error)]
pub enum RepoError {
    #[error("node {0} not found")]  NotFound(NodeId),   // embeds a domain newtype
    #[error("conflict: {0}")]       Conflict(String),
    #[error("validation failed")]   Validation(Vec<FieldError>),
    #[error("aws: {0}")]            Aws(String),
}
```

Map to transport in one place at the edge, not inside the logic:

```rust
fn http_parts(e: &RepoError) -> (u16, &'static str) {
    match e {
        RepoError::NotFound(_)   => (404, "not_found"),
        RepoError::Conflict(_)   => (409, "conflict"),
        RepoError::Validation(_) => (422, "validation"),
        RepoError::Aws(_)        => (502, "upstream"),
    }
}
```

## Generics/`impl Trait` over `dyn`

Prefer monomorphized generics and `impl Trait` at hot boundaries (the closure ports above are generic, so they inline). Reach for `dyn` **only** where you must name/store/erase the type: a boxed future in a type alias (`Pin<Box<dyn Future + Send>>`), a stored callback (`Box<dyn Fn() -> ...>`), or a heterogeneous collection. Use `impl Into<String>` for ergonomic constructor params and `impl Iterator<Item = T>` for returned sequences.

## Common mistakes

- **Bool/Option soup instead of an enum.** Three `Option` fields where one enum with three variants belongs. If two fields are "always set together or never," they're one variant.
- **Validating without changing the type.** `fn check(s: &str) -> bool` then passing the `&str` onward re-opens the question downstream. Parse into a newtype so "valid" is proven by the type.
- **Defining a trait for a single-impl port.** If there's exactly one production implementation and the only other "impl" is a test double, use a closure.
- **`String` errors / `.unwrap()` in the domain.** Model the failure as a variant; unwrap only where the invariant is truly local and proven.
- **`Box<dyn>` by reflex.** If the type is known at the call site, generics inline and cost nothing. Erase only when you must.
- **Sequential awaits on independent work.** Two `.await`s in a row (or `.await` in a `for` loop) where neither uses the other's result → `try_join!` / `try_join_all`.
- **Logic that imports the DB SDK.** The `logic`/`domain` layer should not name `aws_sdk_*`, `sqlx`, etc. If it does, an effect leaked in that should have been a closure parameter.

## Red flags — stop and reconsider the type

- Reaching for a comment to explain which field combinations are "valid" → the type is too loose; split into variants.
- Writing a mock/fake struct that implements a trait with one production impl → should be a closure.
- A test needs a real database, network, or `#[tokio::test]` with live AWS to exercise a *business rule* → the rule isn't isolated behind an injected closure.
- `match` arms that convert your own enum to/from strings by hand → derive `strum::Display`/`EnumString`.
