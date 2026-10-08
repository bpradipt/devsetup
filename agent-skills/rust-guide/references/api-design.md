# API and Function Design

### 3.1 Return `Option` or `Result` — not both inconsistently

When a function can only fail in a limited number of known ways, prefer `Result` over `Option`.

**Don't:**
```rust
fn get_exe_basename() -> Option<String> {
    let exe_name = env::args().next()?;
    let exe_basename = Path::new(&exe_name).file_name()?.to_str()?;
    Some(exe_basename.to_string())
}
```

**Do (if failure is unexpected):**
```rust
fn get_exe_basename() -> Result<String> {
    let exe = env::current_exe().context("couldn't get executable path")?;
    let name = exe.file_name()
        .context("executable has no file name")?
        .to_str()
        .context("executable name is not valid UTF-8")?;
    Ok(name.to_string())
}
```

Also prefer `env::current_exe()` over manually parsing `env::args().next()`.

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

### 3.2 Use the proper type family

Use the strongest `std` type available. Parse into strong types early in the API flow.

| Don't use | Use instead | Why |
|---|---|---|
| `String` for file paths | `PathBuf` / `&Path` | OS path semantics |
| `String` for URLs | `Url` | Validation, components |
| `u64` for duration | `Duration` | Self-documenting, no unit confusion |
| `(usize, usize)` for ranges | `Range<usize>` or `impl RangeBounds` | Idiomatic, flexible |

> **[MS]** M-STRONG-TYPES, M-IMPL-RANGEBOUNDS

### 3.3 Accept `impl AsRef<T>` in function signatures where feasible

For functions that don't need ownership, accept `impl AsRef<T>` for common reference hierarchies:

```rust
// Accepts &str, String, &String, etc.
fn print(x: impl AsRef<str>) {}

// Accepts &Path, PathBuf, &str, String, etc.
fn read_file(x: impl AsRef<Path>) {}

// Accepts &[u8], Vec<u8>, etc.
fn send_network(x: impl AsRef<[u8]>) {}
```

Don't infect struct type parameters with these bounds — keep structs using owned types:

```rust
// Don't
struct User<T: AsRef<str>> { name: T }

// Do
struct User { name: String }
```

> **[MS]** M-IMPL-ASREF

### 3.4 Keep storage/backend details out of upper layers

Database table names, file suffixes, and storage paths are implementation details that should be encapsulated in the storage layer, not leaked to callers.

**Don't:**
```rust
// Upper layer appends storage-specific suffixes
let policy_id = format!("{}{}", policy_id, T::policy_suffix());
```

**Do:** Let the storage backend handle naming internally:
```rust
trait Storage {
    fn get<T: Storable>(&self, id: &str) -> Result<T>;
}

trait Storable {
    fn collection_name() -> &'static str;
    fn suffix() -> Option<&'static str> { None }
}
```

> "A database schema is an implementation detail of an application that a user shouldn't be concerned about. Making it a configuration surface will increase complexity without a good use case."

> **[trustee]** [PR #1131](https://github.com/confidential-containers/trustee/pull/1131)

### 3.5 Don't leak external types in public APIs

Prefer `std` types in public API surfaces over third-party crate types. Any type in your public API becomes part of your contract.

- If avoidable, don't leak third-party types
- Within an umbrella crate, freely leak sibling crate types
- Behind a feature flag, types may be leaked (e.g., `serde`)
- Without a feature, only if there is a substantial benefit

> **[MS]** M-DONT-LEAK-TYPES

### 3.6 Start with restricted APIs, expand later

> "We can iterate from restricted to more flexible configuration options later, when concrete use cases are being requested by users. As long as we don't have a large API/config surface to support, this is easy because it's an internal refactoring. Once an API or config surface is exposed, it's harder to iterate."

**Rule:** Don't expose configuration options that aren't needed yet. Hardcode internal details (table names, directory names) until a real use case demands configurability.

> **[trustee]** [PR #1131](https://github.com/confidential-containers/trustee/pull/1131)

### 3.7 Complex type construction uses builders

Types supporting 4+ initialization permutations should provide builders:

```rust
impl Foo {
    pub fn builder() -> FooBuilder { ... }
}

impl FooBuilder {
    pub fn a(mut self, a: A) -> Self { ... }
    pub fn b(mut self, b: B) -> Self { ... }
    pub fn build(self) -> Foo { ... }
}
```

Conventions:
- Builder for `Foo` is named `FooBuilder`
- Methods are chainable; final method is `.build()`
- Shortcut: `Foo::builder()` — no public `FooBuilder::new()`
- Setter for field `x` is named `x()`, not `set_x()`
- Required parameters go in `builder()`, not as setters

> **[MS]** M-INIT-BUILDER

### 3.8 Prefer regular functions over unrelated associated functions

Associated functions should primarily be for instance creation. Functionality not directly related to a type's receiver shouldn't live in `impl`:

```rust
struct Database {}

impl Database {
    fn new() -> Self {}       // Ok: creates instance
    fn query(&self) {}        // Ok: method with receiver
}

// Not a Database concern — make it a free function
fn check_parameters(p: &str) {}
```

> **[MS]** M-REGULAR-FN
