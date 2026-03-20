# Rust Code Review Guide

> Rust review guide. The compiler catches memory safety, but reviewers still need to judge logic, API design, performance, cancellation safety, and maintainability.

## Contents

- [Ownership and borrowing](#ownership-and-borrowing)
- [Unsafe code review](#unsafe-code-review)
- [Async code](#async-code)
- [Cancellation safety](#cancellation-safety)
- [spawn vs await](#spawn-vs-await)
- [Error handling](#error-handling)
- [Performance](#performance)
- [Trait design](#trait-design)
- [Rust Review Checklist](#rust-review-checklist)

---

## Ownership and borrowing

### Avoid unnecessary `clone()`

```rust
// ❌ clone() as a default escape hatch from the borrow checker
fn bad_process(data: &Data) -> Result<()> {
    let owned = data.clone();  // why clone?
    expensive_operation(owned)
}

// ✅ Ask: is clone required? can we pass a reference?
fn good_process(data: &Data) -> Result<()> {
    expensive_operation(data)
}

// ✅ If clone is required, document why
fn justified_clone(data: &Data) -> Result<()> {
    // Clone needed: value moves into a spawned task
    let owned = data.clone();
    tokio::spawn(async move {
        process(owned).await
    });
    Ok(())
}
```

### `Arc<Mutex<T>>`

```rust
// ❌ Easy to hide shared mutable state you don’t need
struct BadService {
    cache: Arc<Mutex<HashMap<String, Data>>>,  // is sharing really required?
}

// ✅ Single owner when possible
struct GoodService {
    cache: HashMap<String, Data>,
}

// ✅ Finer-grained concurrency when you do need sharing
use dashmap::DashMap;

struct ConcurrentService {
    cache: DashMap<String, Data>,
}
```

### `Cow` (copy-on-write)

```rust
use std::borrow::Cow;

// ❌ Always allocating
fn bad_process_name(name: &str) -> String {
    if name.is_empty() {
        "Unknown".to_string()
    } else {
        name.to_string()  // extra alloc when borrow would do
    }
}

// ✅ Cow avoids extra allocs
fn good_process_name(name: &str) -> Cow<'_, str> {
    if name.is_empty() {
        Cow::Borrowed("Unknown")
    } else {
        Cow::Borrowed(name)
    }
}

// ✅ Allocate only when mutating
fn normalize_name(name: &str) -> Cow<'_, str> {
    if name.chars().any(|c| c.is_uppercase()) {
        Cow::Owned(name.to_lowercase())
    } else {
        Cow::Borrowed(name)
    }
}
```

---

## Unsafe code review

### Minimum bar

```rust
// ❌ unsafe with no safety story — red flag
unsafe fn bad_transmute<T, U>(t: T) -> U {
    std::mem::transmute(t)
}

// ✅ Document invariants for every unsafe API
/// Transmutes `T` to `U`.
///
/// # Safety
///
/// - `T` and `U` must have the same size and alignment
/// - `T` must be a valid bit pattern for `U`
/// - The caller ensures no references to `t` exist after this call
unsafe fn documented_transmute<T, U>(t: T) -> U {
    // SAFETY: Caller guarantees size/alignment match and bit validity
    std::mem::transmute(t)
}
```

### Comments on `unsafe` blocks

```rust
// ❌ unexplained unsafe
fn bad_get_unchecked(slice: &[u8], index: usize) -> u8 {
    unsafe { *slice.get_unchecked(index) }
}

// ✅ SAFETY comment per block
fn good_get_unchecked(slice: &[u8], index: usize) -> u8 {
    debug_assert!(index < slice.len(), "index out of bounds");
    // SAFETY: index < slice.len() checked by debug_assert.
    // Release builds: caller must enforce bounds.
    unsafe { *slice.get_unchecked(index) }
}

// ✅ Wrap unsafe behind a safe API
pub fn checked_get(slice: &[u8], index: usize) -> Option<u8> {
    if index < slice.len() {
        // SAFETY: bounds checked above
        Some(unsafe { *slice.get_unchecked(index) })
    } else {
        None
    }
}
```

### Common `unsafe` patterns

```rust
// ✅ FFI boundary
extern "C" {
    fn external_function(ptr: *const u8, len: usize) -> i32;
}

pub fn safe_wrapper(data: &[u8]) -> Result<i32, Error> {
    // SAFETY: data.as_ptr() valid for data.len() bytes;
    // external_function only reads the buffer.
    let result = unsafe {
        external_function(data.as_ptr(), data.len())
    };
    if result < 0 {
        Err(Error::from_code(result))
    } else {
        Ok(result)
    }
}

// ✅ Hot-path copy
pub fn fast_copy(src: &[u8], dst: &mut [u8]) {
    assert_eq!(src.len(), dst.len(), "slices must be equal length");
    // SAFETY: equal-length slices; dst is exclusive for this write.
    unsafe {
        std::ptr::copy_nonoverlapping(
            src.as_ptr(),
            dst.as_mut_ptr(),
            src.len()
        );
    }
}
```

---

## Async code

### Don’t block the runtime

```rust
// ❌ Blocking in async — stalls other tasks on the same worker
async fn bad_async() {
    let data = std::fs::read_to_string("file.txt").unwrap();
    std::thread::sleep(Duration::from_secs(1));
}

// ✅ Async I/O and timers
async fn good_async() -> Result<String> {
    let data = tokio::fs::read_to_string("file.txt").await?;
    tokio::time::sleep(Duration::from_secs(1)).await;
    Ok(data)
}

// ✅ spawn_blocking for unavoidable blocking work
async fn with_blocking() -> Result<Data> {
    let result = tokio::task::spawn_blocking(|| {
        expensive_cpu_computation()
    }).await?;
    Ok(result)
}
```

### `Mutex` and `.await`

```rust
// ❌ std::sync::Mutex guard held across .await — deadlock risk
async fn bad_lock(mutex: &std::sync::Mutex<Data>) {
    let guard = mutex.lock().unwrap();
    async_operation().await;  // lock held across await
    process(&guard);
}

// ✅ Shrink critical section
async fn good_lock_scoped(mutex: &std::sync::Mutex<Data>) {
    let data = {
        let guard = mutex.lock().unwrap();
        guard.clone()
    };
    async_operation().await;
    process(&data);
}

// ✅ tokio::sync::Mutex if you must await while locked
async fn good_lock_tokio(mutex: &tokio::sync::Mutex<Data>) {
    let guard = mutex.lock().await;
    async_operation().await;
    process(&guard);
}

// 💡 Rule of thumb:
// - std::sync::Mutex: low contention, short sections, no await inside guard
// - tokio::sync::Mutex: need to await while holding the lock
```

### Async trait methods

```rust
// ❌ async_trait: implicit boxing / overhead (older pattern)
#[async_trait]
trait BadRepository {
    async fn find(&self, id: i64) -> Option<Entity>;
}

// ✅ Rust 1.75+: async in traits (RPITIT)
trait Repository {
    async fn find(&self, id: i64) -> Option<Entity>;

    fn find_many(&self, ids: &[i64]) -> impl Future<Output = Vec<Entity>> + Send;
}

// ✅ dyn-friendly: return pinned boxed Future
trait DynRepository: Send + Sync {
    fn find(&self, id: i64) -> Pin<Box<dyn Future<Output = Option<Entity>> + Send + '_>>;
}
```

---

## Cancellation safety

### What it means

```rust
// When a Future is dropped at an .await, what state is left behind?
// Cancel-safe: dropping at any await is OK
// Cancel-unsafe: cancellation can lose data or break invariants

// ❌ Cancel-unsafe sketch
async fn cancel_unsafe(conn: &mut Connection) -> Result<()> {
    let data = receive_data().await;  // if cancelled here...
    conn.send_ack().await;            // ...ack may never run
    Ok(())
}

// ✅ Prefer atomic / transactional steps
async fn cancel_safe(conn: &mut Connection) -> Result<()> {
    let transaction = conn.begin_transaction().await?;
    let data = receive_data().await;
    transaction.commit_with_ack(data).await?;
    Ok(())
}
```

### `select!` and cancellation

```rust
use tokio::select;

// ❌ Dropping a read future can discard partial progress semantics
async fn bad_select(stream: &mut TcpStream) {
    let mut buffer = vec![0u8; 1024];
    loop {
        select! {
            result = stream.read(&mut buffer) => {
                handle_data(&buffer[..result?]);
            }
            _ = tokio::time::sleep(Duration::from_secs(5)) => {
                println!("Timeout");
            }
        }
    }
}

// ✅ Use APIs whose cancellation semantics you understand
async fn good_select(stream: &mut TcpStream) {
    let mut buffer = vec![0u8; 1024];
    loop {
        select! {
            result = stream.read(&mut buffer) => {
                match result {
                    Ok(0) => break,
                    Ok(n) => handle_data(&buffer[..n]),
                    Err(e) => return Err(e),
                }
            }
            _ = tokio::time::sleep(Duration::from_secs(5)) => {
                println!("Timeout, retrying...");
            }
        }
    }
}

// ✅ tokio::pin! for reusable futures in select!
async fn pinned_select() {
    let sleep = tokio::time::sleep(Duration::from_secs(10));
    tokio::pin!(sleep);

    loop {
        select! {
            _ = &mut sleep => {
                println!("Timer elapsed");
                break;
            }
            data = receive_data() => {
                process(data).await;
            }
        }
    }
}
```

### Document cancellation

```rust
/// Reads a complete message from the stream.
///
/// # Cancel safety
///
/// **Not** cancel safe. If cancelled mid-read, partial bytes may be lost
/// and stream state may be inconsistent. Prefer `read_message_cancel_safe`
/// when callers may drop the future early.
async fn read_message(stream: &mut TcpStream) -> Result<Message> {
    let len = stream.read_u32().await?;
    let mut buffer = vec![0u8; len as usize];
    stream.read_exact(&mut buffer).await?;
    Ok(Message::from_bytes(&buffer))
}

/// Cancel-safe message read.
///
/// # Cancel safety
///
/// Cancel safe: partial data stays in an internal buffer for the next call.
async fn read_message_cancel_safe(reader: &mut BufferedReader) -> Result<Message> {
    reader.read_message_buffered().await
}
```

---

## spawn vs await

### When to `spawn`

```rust
// ❌ spawn + immediate await — extra overhead, no real parallelism
async fn bad_unnecessary_spawn() {
    let handle = tokio::spawn(async {
        simple_operation().await
    });
    handle.await.unwrap();
}

// ✅ await directly
async fn good_direct_await() {
    simple_operation().await;
}

// ✅ True parallelism
async fn good_parallel_spawn() {
    let task1 = tokio::spawn(fetch_from_service_a());
    let task2 = tokio::spawn(fetch_from_service_b());
    let (result1, result2) = tokio::try_join!(task1, task2)?;
}

// ✅ Detached background work (with care: log panics, lifecycle)
async fn good_background_spawn() {
    tokio::spawn(async {
        cleanup_old_sessions().await;
        log_metrics().await;
    });
    handle_request().await;
}
```

### `'static` on spawned futures

```rust
// ❌ Borrowed data in spawn
async fn bad_spawn_borrow(data: &Data) {
    tokio::spawn(async {
        process(data).await;  // Error: `data` not 'static
    });
}

// ✅ Clone into the task
async fn good_spawn_clone(data: &Data) {
    let owned = data.clone();
    tokio::spawn(async move {
        process(&owned).await;
    });
}

// ✅ Arc for shared ownership
async fn good_spawn_arc(data: Arc<Data>) {
    let data = Arc::clone(&data);
    tokio::spawn(async move {
        process(&data).await;
    });
}

// ✅ Scoped crates when you must borrow (know the limits)
async fn good_scoped_spawn(data: &Data) {
    async_scoped::scope(|s| async {
        s.spawn(async {
            process(data).await;
        });
    }).await;
}
```

### `JoinHandle` errors

```rust
// ❌ Ignoring join outcome
async fn bad_ignore_spawn_error() {
    let handle = tokio::spawn(async {
        risky_operation().await
    });
    let _ = handle.await;
}

// ✅ Handle Ok / Err / panic
async fn good_handle_spawn_error() -> Result<()> {
    let handle = tokio::spawn(async {
        risky_operation().await
    });

    match handle.await {
        Ok(Ok(result)) => {
            process_result(result);
            Ok(())
        }
        Ok(Err(e)) => Err(e.into()),
        Err(join_err) => {
            if join_err.is_panic() {
                error!("Task panicked: {:?}", join_err);
            }
            Err(anyhow!("Task failed: {}", join_err))
        }
    }
}
```

### Structured concurrency vs spawn

```rust
// ✅ prefer try_join! / join! in one scope
async fn structured_concurrency() -> Result<(A, B, C)> {
    tokio::try_join!(
        fetch_a(),
        fetch_b(),
        fetch_c()
    )
}

// ✅ If you spawn, own shutdown
struct TaskManager {
    handles: Vec<JoinHandle<()>>,
}

impl TaskManager {
    async fn shutdown(self) {
        for handle in self.handles {
            if let Err(e) = handle.await {
                error!("Task failed during shutdown: {}", e);
            }
        }
    }

    async fn abort_all(self) {
        for handle in self.handles {
            handle.abort();
        }
    }
}
```

---

## Error handling

### Library vs application errors

```rust
// ❌ anyhow in a library — callers can’t match variants
pub fn parse_config(s: &str) -> anyhow::Result<Config> { ... }

// ✅ thiserror in libs; anyhow common in apps
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("invalid syntax at line {line}: {message}")]
    Syntax { line: usize, message: String },
    #[error("missing required field: {0}")]
    MissingField(String),
    #[error(transparent)]
    Io(#[from] std::io::Error),
}

pub fn parse_config(s: &str) -> Result<Config, ConfigError> { ... }
```

### Preserve context

```rust
// ❌ Loses root cause
fn bad_error() -> Result<()> {
    operation().map_err(|_| anyhow!("failed"))?;
    Ok(())
}

// ✅ anyhow context / tracing-error
fn good_error() -> Result<()> {
    operation().context("failed to perform operation")?;
    Ok(())
}

fn good_error_lazy() -> Result<()> {
    operation()
        .with_context(|| format!("failed to process file: {}", filename))?;
    Ok(())
}
```

### Error type design

```rust
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("database error")]
    Database(#[source] sqlx::Error),

    #[error("network error: {message}")]
    Network {
        message: String,
        #[source]
        source: reqwest::Error,
    },

    #[error("validation failed: {0}")]
    Validation(String),
}

impl From<sqlx::Error> for ServiceError {
    fn from(err: sqlx::Error) -> Self {
        ServiceError::Database(err)
    }
}
```

---

## Performance

### Avoid gratuitous `collect()`

```rust
// ❌ Intermediate Vec
fn bad_sum(items: &[i32]) -> i32 {
    items.iter()
        .filter(|x| **x > 0)
        .collect::<Vec<_>>()
        .iter()
        .sum()
}

// ✅ Single iterator pass
fn good_sum(items: &[i32]) -> i32 {
    items.iter().filter(|x| **x > 0).copied().sum()
}
```

### Strings

```rust
// ❌ Repeated realloc in a loop
fn bad_concat(items: &[&str]) -> String {
    let mut s = String::new();
    for item in items {
        s = s + item;
    }
    s
}

fn good_concat(items: &[&str]) -> String {
    items.join("")
}

fn good_concat_capacity(items: &[&str]) -> String {
    let total_len: usize = items.iter().map(|s| s.len()).sum();
    let mut result = String::with_capacity(total_len);
    for item in items {
        result.push_str(item);
    }
    result
}

use std::fmt::Write;

fn good_concat_write(items: &[&str]) -> String {
    let mut result = String::new();
    for item in items {
        write!(result, "{}", item).unwrap();
    }
    result
}
```

### Fewer allocations

```rust
// ❌ collect just to call is_empty
fn bad_check_any(items: &[Item]) -> bool {
    let filtered: Vec<_> = items.iter()
        .filter(|i| i.is_valid())
        .collect();
    !filtered.is_empty()
}

// ✅ Iterator combinator
fn good_check_any(items: &[Item]) -> bool {
    items.iter().any(|i| i.is_valid())
}

// ❌ heap alloc for a static literal
fn bad_static() -> String {
    String::from("error message")
}

// ✅ &'static str when possible
fn good_static() -> &'static str {
    "error message"
}
```

---

## Trait design

### Avoid abstraction for its own sake

```rust
// ❌ “interface all the things”
trait Processor { fn process(&self); }
trait Handler { fn handle(&self); }
trait Manager { fn manage(&self); }

// ✅ Traits where polymorphism pays off
struct DataProcessor {
    config: Config,
}

impl DataProcessor {
    fn process(&self, data: &Data) -> Result<Output> {
        // concrete implementation
    }
}
```

### Trait objects vs generics

```rust
// ❌ dyn when monomorphization would do
fn bad_process(handler: &dyn Handler) {
    handler.handle();
}

// ✅ Generic — static dispatch, inlinable
fn good_process<H: Handler>(handler: &H) {
    handler.handle();
}

// ✅ Heterogeneous collections need trait objects
fn store_handlers(handlers: Vec<Box<dyn Handler>>) {
}

// ✅ impl Trait return
fn create_handler() -> impl Handler {
    ConcreteHandler::new()
}
```

---

## Rust Review Checklist

### What the compiler won’t check

**Correctness**
- [ ] Edge cases covered
- [ ] State machine transitions complete
- [ ] Races ruled out or documented

**API design**
- [ ] Hard to misuse publicly
- [ ] Types express intent
- [ ] Error granularity fits callers

### Ownership and borrowing

- [ ] Every `clone()` justified and noted
- [ ] `Arc<Mutex<T>>` only when shared mutable state is real
- [ ] `RefCell` has a clear reason
- [ ] Lifetimes as simple as possible
- [ ] `Cow` used to cut clones/allocs where it helps

### Unsafe (highest scrutiny)

- [ ] SAFETY comment on every `unsafe` block
- [ ] `# Safety` on every `unsafe fn`
- [ ] Explains *why* safe, not only *what* it does
- [ ] Invariants listed for maintainers
- [ ] Unsafe surface minimized
- [ ] Safer alternative considered

### Async / concurrency

- [ ] No blocking (`std::fs`, `thread::sleep`, …) on async runtimes
- [ ] No `std::sync` locks held across `.await` (unless you mean it)
- [ ] Spawned futures are `'static` or scoped correctly
- [ ] Consistent lock ordering
- [ ] Sensible channel buffer sizes

### Cancellation safety

- [ ] Futures in `select!` are cancel-safe or documented otherwise
- [ ] Public async APIs document cancel behavior
- [ ] Cancellation doesn’t lose invariants / data silently
- [ ] `tokio::pin!` where a future is reused across polls

### spawn vs await

- [ ] `spawn` only for real parallelism or intentional background work
- [ ] Simple path: `await` without wrapping
- [ ] `JoinHandle` results handled (incl. panics)
- [ ] Lifecycle / shutdown strategy for long-running tasks
- [ ] Prefer `join!` / `try_join!` for structured concurrency

### Errors

- [ ] Libraries: `thiserror` (or similar) for typed errors
- [ ] Apps: `anyhow` / `eyre` + `.context()`
- [ ] No `unwrap`/`expect` on fallible production paths
- [ ] Messages actionable for operators
- [ ] `#[must_use]` results not dropped silently
- [ ] `#[source]` preserves chains

### Performance

- [ ] No pointless `collect()`
- [ ] Large values by reference
- [ ] String building: `with_capacity`, `join`, `write!`
- [ ] `impl Trait` vs `Box<dyn Trait>` chosen deliberately
- [ ] Hot paths allocation-aware
- [ ] `Cow` to reduce clones where profiling says so

### Quality

- [ ] `cargo clippy` clean (or waivers documented)
- [ ] `cargo fmt`
- [ ] Public items documented
- [ ] Tests for tricky boundaries
- [ ] Examples on important APIs
