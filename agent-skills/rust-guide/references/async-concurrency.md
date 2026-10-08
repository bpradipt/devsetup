# Async and Concurrency

### 10.1 Public types should be `Send`

Public types should be `Send` for compatibility with `tokio` and other work-stealing runtimes. All futures must be `Send`.

Assert `Send` in tests for key entry points:

```rust
async fn process_request() { /* ... */ }

fn assert_send<T: Send>(_: T) {}

#[test]
fn future_is_send() {
    _ = assert_send(process_request());
}
```

Holding `!Send` types (like `Rc`) across `.await` points prevents the future from being `Send`:

```rust
// BAD: Rc held across .await makes future !Send
async fn foo() {
    let rc = Rc::new(123);
    read_file("foo.txt").await;  // .await while rc is alive
    dbg!(rc);
}
```

> **[MS]** M-TYPES-SEND

### 10.2 Long-running tasks should have yield points

If performing long-running CPU-bound computations in async code, add `yield_now().await` to avoid starving other tasks:

```rust
async fn process_items(items: &[Item]) {
    for item in items {
        decompress(item);       // CPU-bound work
        yield_now().await;      // Let other tasks run
    }
}
```

Rule of thumb: perform 10-100 microseconds of CPU-bound work between yield points.

> **[MS]** M-YIELD-POINTS

### 10.3 Services use `Arc<Inner>` for shared-ownership `Clone`

Heavyweight service types should implement `Clone` via `Arc<Inner>`:

```rust
struct ServiceInner {
    // ... expensive resources ...
}

#[derive(Clone)]
pub struct Service {
    inner: Arc<ServiceInner>,
}

impl Service {
    pub fn new() -> Self {
        Self { inner: Arc::new(ServiceInner::new()) }
    }
}
```

This allows services to be cheaply cloned and shared across handlers without wrapping in `Arc<Mutex<>>` at the call site.

> **[MS]** M-SERVICES-CLONE
