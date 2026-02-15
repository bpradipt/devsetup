# Rust Coding Guidelines

Guidelines merged from [trustee](https://github.com/confidential-containers/trustee) PR review comments and [Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/).

Sources are tagged: **[trustee]** for project-specific PR feedback, **[MS]** for Microsoft guidelines.

---

## 1. Error Handling

### 1.1 Avoid `unwrap()` in application code — use `?` or `.expect()` with context

**Don't:**
```rust
trustee_run(config_file, &trustee_home_dir).await.unwrap();
```
With `unwrap()` the user gets an unhelpful panic message:
```
thread 'main' panicked at tools/trustee/src/cli.rs:126:18:
called `Result::unwrap()` on an `Err` value: refusing to overwrite file: "/home/pvl/.trustee/https_key.pem"
```

**Do:**
```rust
trustee_run(config_file, &trustee_home_dir).await?;
```
Or if you must surface the error in a CLI context:
```rust
.map_err(|e| Error::raw(clap::error::ErrorKind::InvalidValue, format!("{}\n", e)))?
```
This produces a clean error:
```
error: refusing to overwrite file: "/home/pvl/.trustee/https_key.pem"
```

**Rule:** If a function already returns `Result`, propagate errors with `?` instead of calling `.unwrap()`. Use `.expect("reason")` only when failure is truly impossible and you want to document why.

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

### 1.2 Panics are for programming bugs, not recoverable errors

Panics mean "stop the program." They are not exceptions and should not be used for control flow. Valid reasons to panic:

- Encountering a programming bug: `x.expect("invariant: queue is never empty")`
- Const contexts: `const { foo.unwrap() }`
- Encountering a poisoned lock (signals another thread already panicked)

**Don't:**
```rust
fn process(input: &str) -> Result<(), MyError> {
    if input.is_empty() {
        panic!("empty input");  // This is a recoverable error, not a bug
    }
    // ...
}
```

**Do:**
```rust
fn process(input: &str) -> Result<(), MyError> {
    if input.is_empty() {
        bail!("input must not be empty");  // Recoverable: let caller decide
    }
    // ...
}
```

Contract violations that indicate programming errors (not user errors) should panic. Make it correct by construction using the type system to avoid the panic path entirely.

> **[MS]** M-PANIC-IS-STOP, M-PANIC-ON-BUG

### 1.3 Use `anyhow` for applications; structured errors for libraries

**Applications** (binaries, CLI tools): use `anyhow`, `eyre`, or similar. Once selected, use it consistently — don't mix multiple application error types.

```rust
use anyhow::{bail, Context, Result};

pub async fn cli_default() -> Result<()> {
    let config = load_config(&path)
        .context("failed to load configuration")?;
    // ...
}
```

**Libraries** (crates consumed by others): create situation-specific error structs. Don't use a single global error enum for unrelated operations. Don't mix `anyhow` and `thiserror` in the same module.

```rust
// Prefer separate error types for separate operations
fn download_iso() -> Result<(), DownloadError> {}
fn start_vm() -> Result<(), VmError> {}

// Not a single enum for everything
fn download_iso() -> Result<(), GlobalError> {}
fn start_vm() -> Result<(), GlobalError> {}
```

If using an inner `ErrorKind` enum, don't expose it directly — expose `is_xxx()` methods instead:

```rust
pub struct HttpError {
    kind: ErrorKind,      // private
    backtrace: Backtrace,
}

impl HttpError {
    pub fn is_io(&self) -> bool { matches!(self.kind, ErrorKind::Io(_)) }
    pub fn is_protocol(&self) -> bool { matches!(self.kind, ErrorKind::Protocol) }
}
```

> **[trustee]** [PR #851](https://github.com/confidential-containers/trustee/pull/851) | **[MS]** M-APP-ERROR, M-ERRORS-CANONICAL-STRUCTS

### 1.4 Use `map_err` + `?` instead of nested `match` on `Result`

**Don't:**
```rust
let secret_data: HashMap<String, String> = match kv1::get(&self.client, &self.mount_path, &vault_path).await {
    Ok(data) => data,
    Err(e) if e.to_string().contains("status code 404") => {
        return Err(VaultError::SecretNotFound { path: vault_path }.into())
    }
    Err(e) => {
        return Err(VaultError::VaultApiError {
            path: vault_path,
            source: e.into(),
        }.into())
    }
};
```

**Do:**
```rust
let secret_data: HashMap<String, String> = kv1::get(&self.client, &self.mount_path, &vault_path)
    .await
    .map_err(|e| {
        if e.to_string().contains("status code 404") {
            VaultError::SecretNotFound { path: vault_path.clone() }.into()
        } else {
            VaultError::VaultApiError {
                path: vault_path.clone(),
                source: e.into(),
            }.into()
        }
    })?;
```

> **[trustee]** rust_guidelines.md

### 1.5 Use `.context()` to convert `Option` to `Result`

**Don't:**
```rust
let data_value = secret_data.get("data");
if let Some(value) = data_value {
    Ok(value.as_bytes().to_vec())
} else {
    let available_keys = secret_data.keys().map(|k| k.to_string()).collect();
    Err(VaultError::DataKeyMissing { path: vault_path, available_keys }.into())
}
```

**Do:**
```rust
secret_data.get("data")
    .map(|value| value.as_bytes().to_vec())
    .context({
        let available_keys = secret_data.keys().map(|k| k.to_string()).collect();
        VaultError::DataKeyMissing { path: vault_path, available_keys }
    })
```

> **[trustee]** rust_guidelines.md

### 1.6 Be strict with error handling in security-sensitive code

**Don't** silently drop or ignore unexpected data:
```rust
// Silently filtering out policies without a suffix
let policies = policies
    .into_iter()
    .filter_map(|policy| {
        policy.strip_suffix(T::policy_suffix())
            .map(|policy| policy.to_string())
    })
    .collect();
```

**Do** bail on unexpected input:
> "Being forgiving (silently dropping) on faulty input data is almost never a good idea, especially for security-sensitive applications."

```rust
let policies: Result<Vec<String>> = policies
    .into_iter()
    .map(|policy| {
        policy.strip_suffix(T::policy_suffix())
            .map(|p| p.to_string())
            .ok_or_else(|| anyhow!("policy '{}' missing expected suffix", policy))
    })
    .collect();
let policies = policies?;
```

> **[trustee]** [PR #1131](https://github.com/confidential-containers/trustee/pull/1131)

---

## 2. Control Flow and Pattern Matching

### 2.1 Use `if let` when you only care about one variant

**Don't:**
```rust
match expected_report_data {
    ReportData::Value(expected_report_data) => {
        verify_nonce(&ev.quote, expected_report_data)?;
    }
    ReportData::NotProvided => {}
}
```

**Do:**
```rust
if let ReportData::Value(report_data) = expected_report_data {
    verify_nonce(&ev.quote, report_data)?;
}
```

**When to use `if let`:**
- You only care about one variant
- Other variants should be ignored
- The action is simple

**When to use `match`:**
- You need to handle multiple variants differently
- You want exhaustive checking
- The logic is complex

> **[trustee]** [PR #851](https://github.com/confidential-containers/trustee/pull/851)

### 2.2 Don't use `match true/false` — use `if/else`

There is a clippy rule enforcing this.

**Don't:**
```rust
match entries.len() > config.max_trusted_ak_keys {
    true => {
        warn!("Number of trusted AK keys ({}) exceeds the limit ({}).",
              entries.len(), config.max_trusted_ak_keys);
    }
    false => {}
}
```

**Do:**
```rust
if entries.len() > config.max_trusted_ak_keys {
    warn!("Number of trusted AK keys ({}) exceeds the limit ({}).",
          entries.len(), config.max_trusted_ak_keys);
}
```

> **[trustee]** [PR #851](https://github.com/confidential-containers/trustee/pull/851)

### 2.3 Reduce indentation with early returns and guard clauses

**Don't:**
```rust
for entry in entries.into_iter().take(config.max_trusted_ak_keys) {
    let path = entry.path();
    if path.is_file() {
        // ... deeply nested logic ...
    }
}
```

**Do:**
```rust
for entry in entries.into_iter().take(config.max_trusted_ak_keys) {
    let path = entry.path();
    if !path.is_file() {
        continue;
    }
    // ... flat logic ...
}
```

> **[trustee]** [PR #851](https://github.com/confidential-containers/trustee/pull/851)

### 2.4 Use `let-else` for early returns

```rust
let Some(keys_dir) = config.trusted_ak_keys_dir else {
    return Ok(Self { trusted_ak_hashes });
};

let Some(extension) = path.extension() else {
    continue;
};
```

> **[trustee]** [PR #851](https://github.com/confidential-containers/trustee/pull/851)

### 2.5 Invert conditions for clearer guard clauses

**Don't:**
```rust
match self.trusted_ak_hashes.contains(&ak_public_hash) {
    true => {},
    false => return Err(TpmVerifierError::UntrustedAkKey.into()),
}
```

**Do:**
```rust
if !self.trusted_ak_hashes.contains(&ak_public_hash) {
    return Err(TpmVerifierError::UntrustedAkKey.into());
}
```

> **[trustee]** [PR #851](https://github.com/confidential-containers/trustee/pull/851)

---

## 3. API and Function Design

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

---

## 4. Testing

### 4.1 Use `rstest` for parameterized tests

```rust
#[rstest]
#[case("test_data/configs/coco-as-grpc-1.toml", KbsConfig {
    attestation_token: AttestationTokenVerifierConfig {
        trusted_certs_paths: vec!["/etc/ca".into(), "/etc/ca2".into()],
        // ...
    },
    // ...
})]
#[case("test_data/configs/coco-as-builtin-1.toml", KbsConfig {
    // ...
})]
fn test_config_parsing(#[case] config_path: &str, #[case] expected: KbsConfig) {
    let config = KbsConfig::try_from(Path::new(config_path)).unwrap();
    assert_eq!(config, expected);
}
```

> **[trustee]** [kbs/src/config.rs](https://github.com/confidential-containers/trustee/blob/main/kbs/src/config.rs#L143-L477)

### 4.2 Include unit tests with evidence fixtures

> "It would be good to have a unit-test with an evidence fixture like in other verifiers (see `deps/verifier/test_data`) to avoid breakage when refactoring."

**Rule:** Always include test data fixtures for verifier/parser code to prevent regressions.

> **[trustee]** [PR #851](https://github.com/confidential-containers/trustee/pull/851)

### 4.3 Ensure tests cover feature flags

If code is gated behind a feature flag, ensure CI tests exercise it:
```rust
#[cfg(feature = "intel-trust-authority-as")]
```

> "To reproduce, I guess you have to enable the feature `intel-trust-authority-as`."

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

### 4.4 Make I/O and system calls mockable

Any user-facing type doing I/O or system calls with side effects should be mockable. This includes file/network access, clocks, entropy sources, and seeds.

```rust
impl Library {
    pub fn new() -> Self { ... }

    #[cfg(feature = "test-util")]
    pub fn new_mocked() -> (Self, MockCtrl) { ... }
}
```

Gate test utilities behind a `test-util` feature flag to prevent production builds from bypassing safety:

```rust
impl HttpClient {
    pub fn get() { ... }

    #[cfg(feature = "test-util")]
    pub fn bypass_certificate_checks() { ... }
}
```

> **[MS]** M-MOCKABLE-SYSCALLS, M-TEST-UTIL

---

## 5. Serialization and Configuration

### 5.1 Use `#[serde(default)]` deliberately

Only add `#[serde(default)]` when a field genuinely has a meaningful default. Don't add it just to make tests pass — it can mask missing configuration.

```rust
pub struct IntelTrustAuthorityConfig {
    pub api_key: String,
    pub certs_file: String,
    pub allow_unmatched_policy: Option<bool>,
    #[serde(default)]  // Only if this field is truly optional
    pub policy_ids: Vec<String>,
}
```

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

### 5.2 Use `Default` trait with `impl Default` for config structs

When removing explicit defaults from config builder code, make sure `impl Default` provides the same values.

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

---

## 6. Logging and Observability

### 6.1 Add logging for configuration used at startup

```rust
info!("Using config: {:?}", config);
debug!("Detailed config: {:#?}", config);
```

> "Can we add a log here like `info!` or `debug!` to show the config actually used."

> **[trustee]** [PR #851](https://github.com/confidential-containers/trustee/pull/851)

### 6.2 Use string constants for environment variable names

**Don't:**
```rust
if let Ok(dev) = env::var("AA_TPM_DEVICE") {
    log::info!("TPM device detected from AA_TPM_DEVICE env: {}", dev);
}
```

**Do:**
```rust
const AA_TPM_DEVICE_ENV: &str = "AA_TPM_DEVICE";

if let Ok(dev) = env::var(AA_TPM_DEVICE_ENV) {
    log::info!("TPM device detected from {} env: {}", AA_TPM_DEVICE_ENV, dev);
}
```

> **[trustee]** rust_guidelines.md

### 6.3 Use structured logging with named events

Avoid string formatting in log calls — it causes allocations. Use message templates with named properties and hierarchical dot-notation event names:

**Don't:**
```rust
tracing::info!("file opened: {}", path);
```

**Do:**
```rust
event!(
    name: "file.open.success",
    Level::INFO,
    file.path = path.display(),
    "file opened: {{file.path}}",
);
```

Name events using `<component>.<operation>.<state>` pattern for filtering.

> **[MS]** M-LOG-STRUCTURED

### 6.4 Document magic values with named constants

Hardcoded values must be named constants with comments explaining why the value was chosen:

**Don't:**
```rust
wait_timeout(60 * 60 * 24).await
```

**Do:**
```rust
/// Timeout long enough for the upstream server to finish processing.
const UPSTREAM_SERVER_TIMEOUT: Duration = Duration::from_secs(60 * 60 * 24);

wait_timeout(UPSTREAM_SERVER_TIMEOUT).await
```

> **[MS]** M-DOCUMENTED-MAGIC

### 6.5 Redact sensitive data in logs

Never log sensitive data in plain text — email addresses, file paths revealing identity, contents with PII, or temp paths with session IDs:

```rust
// Don't
event!(Level::INFO, user.email = user.email, "user {{user.email}}");

// Do
event!(Level::INFO, user.email.redacted = redact_email(user.email), "user {{user.email.redacted}}");
```

> **[MS]** M-LOG-STRUCTURED

---

## 7. Code Organization

### 7.1 Keep unrelated changes out of PRs

> "Is this change actually required? It seems isolated from the rest of the commit."

**Rule:** Each PR should be focused. Unrelated changes (even small ones like adding `#[serde(default)]`) should go in separate commits or PRs.

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

### 7.2 Names are free of weasel words

Avoid names like `Service`, `Manager`, `Factory` that don't add meaningful information:

- `BookingService` -> `Bookings` or `BookingDispatcher`
- `FooManager` -> name after what it actually does
- `FooFactory` -> `FooBuilder` (canonical Rust name)

> **[MS]** M-CONCISE-NAMES

### 7.3 Name enum variants descriptively

> "May want to make this enum variant more explicit in terms of what it is running."

```rust
// Don't
enum Commands {
    Run { ... },
}

// Do
enum Commands {
    /// Launch the Trustee server
    RunServer { ... },
}
```

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

### 7.4 Use `std::env::current_exe()` instead of `env::args().next()`

**Don't:**
```rust
let exe_name = env::args().next()?;
let exe_basename = Path::new(&exe_name).file_name()?.to_str()?;
```

**Do:**
```rust
let exe_basename = env::current_exe()?
    .file_name()
    .expect("couldn't get executable's name")
    .to_str()
    .expect("executable name is not valid UTF-8")
    .to_string();
```

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

### 7.5 Document algorithm choices

> "Maybe document the choice of the key algo somewhere (in the CLI help would be sufficient)."

When using specific cryptographic algorithms (Ed25519, RSA-2048, etc.), document why that algorithm was chosen.

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

### 7.6 If in doubt, split the crate

Err toward having too many crates rather than too few. This improves compile times and prevents cyclic dependencies. If a submodule can be used independently, move it to a separate crate.

Crate splits may lose `pub(crate)` access — this is often desirable, prompting more flexible abstractions.

> **[MS]** M-SMALLER-CRATES

### 7.7 Don't glob re-export items

**Don't:**
```rust
pub use foo::*;  // May export more than intended
```

**Do:**
```rust
pub use foo::{A, B, C};
```

Exception: platform-specific HAL re-exports where the entire module is the contract:
```rust
#[cfg(target_os = "linux")]
pub use linux::*;
```

> **[MS]** M-NO-GLOB-REEXPORTS

---

## 8. Idiomatic Patterns

### 8.1 Use `inspect_err` for side effects on errors

**Don't:**
```rust
match backend.write_secret_resource(resource_desc.clone(), test_data).await {
    Ok(_) => { println!("Success"); }
    Err(e) => { println!("Failed: {}", e); }
}
```

**Do:**
```rust
let _ = backend
    .write_secret_resource(resource_desc.clone(), test_data)
    .await
    .inspect_err(|e| {
        println!("Write failed (may be expected if Vault server isn't configured): {}", e)
    })
    .expect("Failed to write secret resource");
```

> **[trustee]** rust_guidelines.md

### 8.2 Prefer `Option` return type for "already exists" semantics

Consider returning `Option<OldValue>` instead of a custom enum when checking for existing values (similar to `HashMap::insert`):

```rust
// std HashMap returns Some(old_value) if key existed
fn set(&self, key: &str, value: &[u8]) -> Option<Vec<u8>>;
```

Alternatively, just return `Ok(())` and log if the key already exists, rather than introducing a custom `SetResult` enum when callers don't need to distinguish.

> **[trustee]** [PR #1131](https://github.com/confidential-containers/trustee/pull/1131)

### 8.3 Use `if let` instead of `match` with empty catch-all arm

**Don't:**
```rust
match &mut config.attestation_service.attestation_service {
    CoCoASBuiltIn(as_config) => {
        // ... do stuff ...
    }
    _ => {}
}
```

**Do:**
```rust
if let CoCoASBuiltIn(as_config) = &mut config.attestation_service.attestation_service {
    // ... do stuff ...
}
```

> **[trustee]** [PR #851](https://github.com/confidential-containers/trustee/pull/851)

### 8.4 Public types implement `Debug`

All public types should implement `Debug`. Types holding sensitive data must use a custom implementation that hides the data:

```rust
struct UserSecret(String);

impl Debug for UserSecret {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "UserSecret(...)")
    }
}

#[test]
fn debug_does_not_leak_secret() {
    let key = "552d3454-d0d5-445d-ab9f-ef2ae3a8896a";
    let secret = UserSecret(key.to_string());
    let rendered = format!("{:?}", secret);
    assert!(!rendered.contains(key));
}
```

> **[MS]** M-PUBLIC-DEBUG

### 8.5 Follow upstream naming conventions

- **Conversions:** `as_` (cheap ref-to-ref), `to_` (expensive conversion), `into_` (ownership transfer)
- **Getters:** follow Rust convention (no `get_` prefix for simple getters)
- **Constructors:** static inherent methods; always provide `Foo::new()` even if `Foo::default()` exists
- **Common traits:** eagerly implement `Clone`, `Debug`, `Default`, `PartialEq`, `Eq`, `Hash` where appropriate

> **[MS]** M-UPSTREAM-GUIDELINES (C-CONV, C-GETTER, C-CTOR, C-COMMON-TRAITS)

---

## 9. Safety and Soundness

### 9.1 Unsafe needs a reason and should be avoided

Valid reasons to use `unsafe`:
1. **Novel abstractions** — new smart pointers, allocators (must pass Miri)
2. **Performance** — e.g., `.get_unchecked()` (must benchmark first)
3. **FFI and platform calls**

**Don't** use `unsafe` to:
- Shorten safe programs
- Bypass `Send`/`Sync` bounds
- Bypass lifetime requirements via `transmute`
- Mark dangerous-but-safe functions (use documentation instead)

```rust
// Valid: misuse causes UB
unsafe fn print_string(x: *const String) { }

// Invalid: dangerous but not UB
// Just document the danger, don't mark unsafe
fn delete_database() { }
```

> **[MS]** M-UNSAFE, M-UNSAFE-IMPLIES-UB

### 9.2 All code must be sound

Unsound code is never acceptable. A function is unsound if any calling pattern — even theoretical — causes undefined behavior:

```rust
// UNSOUND: "safely" converts types
fn unsound_ref<T>(x: &T) -> &u128 {
    unsafe { std::mem::transmute(x) }
}

// UNSOUND: blanket Send impl
struct AlwaysSend<T>(T);
unsafe impl<T> Send for AlwaysSend<T> {}
```

If you cannot safely encapsulate something, expose `unsafe` functions and document what the caller must uphold.

> **[MS]** M-UNSOUND

---

## 10. Async and Concurrency

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

---

## 11. Static Verification

### 11.1 Use `#[expect]` instead of `#[allow]` for lint overrides

`#[expect]` warns if the suppressed lint wasn't actually triggered, preventing stale overrides from accumulating:

```rust
#[expect(clippy::unused_async, reason = "API is fixed, will use I/O in next iteration")]
pub async fn ping_server() {
    // Stubbed out for now
}
```

> **[MS]** M-LINT-OVERRIDE-EXPECT

### 11.2 Enable recommended lints

**Compiler lints** in `Cargo.toml`:
```toml
[lints.rust]
ambiguous_negative_literals = "warn"
missing_debug_implementations = "warn"
redundant_imports = "warn"
redundant_lifetimes = "warn"
trivial_numeric_casts = "warn"
unsafe_op_in_unsafe_fn = "warn"
unused_lifetimes = "warn"
```

**Clippy lint categories:**
```toml
[lints.clippy]
cargo = { level = "warn", priority = -1 }
complexity = { level = "warn", priority = -1 }
correctness = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
perf = { level = "warn", priority = -1 }
style = { level = "warn", priority = -1 }
suspicious = { level = "warn", priority = -1 }
```

**Key restriction lints:**
```toml
map_err_ignore = "warn"
undocumented_unsafe_blocks = "warn"
string_to_string = "warn"
unused_result_ok = "warn"
clone_on_ref_ptr = "warn"
```

> **[MS]** M-STATIC-VERIFICATION

### 11.3 Use static verification tools

- **Rustfmt** — consistent formatting
- **Clippy** — lint categories above
- **cargo-audit** — dependency vulnerability scanning
- **cargo-hack** — validate all feature combinations compile
- **cargo-udeps** — detect unused dependencies
- **Miri** — validate unsafe code correctness

> **[MS]** M-STATIC-VERIFICATION

---

## 12. Resilience

### 12.1 Avoid statics for state that must be consistent

Libraries should avoid `static` and thread-local items if a consistent view is relevant for correctness. Rust may link multiple versions of the same crate independently, each with its own statics — this silently duplicates state.

Statics for performance optimization only (caches, pools) are acceptable.

> **[MS]** M-AVOID-STATICS

### 12.2 Features must be additive

All library features must be additive — any combination must compile on the current platform:

- Don't introduce a `no-std` feature; use `std` instead (default-on)
- Adding feature `foo` must not disable or modify existing public items
- Features must not rely on other features being manually enabled

> **[MS]** M-FEATURES-ADDITIVE

---

## Summary Checklist

**Error Handling:**
- [ ] No `unwrap()` in application code (use `?`, `.expect()`, or `.context()`)
- [ ] Panics only for programming bugs, not recoverable errors
- [ ] `anyhow` for applications; structured error types for libraries
- [ ] `map_err` + `?` instead of nested `match` on `Result`
- [ ] Strict validation of inputs in security-sensitive code

**Control Flow:**
- [ ] `if let` when handling a single enum variant; `match` for multiple
- [ ] `if/else` for boolean conditions, never `match true/false`
- [ ] Guard clauses with early `return`/`continue` to reduce nesting
- [ ] `let-else` for early exits from `Option`/`Result`

**API Design:**
- [ ] Strong types (`PathBuf`, `Duration`) over primitives
- [ ] `impl AsRef<T>` in function signatures for flexibility
- [ ] Don't leak third-party types in public APIs
- [ ] Builders for types with 4+ initialization permutations
- [ ] Start with restricted APIs, expand when use cases arise

**Testing:**
- [ ] `rstest` for parameterized tests with evidence fixtures
- [ ] Feature-flagged code tested in CI
- [ ] I/O and system calls mockable via `test-util` feature

**Logging:**
- [ ] Structured logging with named events
- [ ] Named constants for magic values and env vars
- [ ] Sensitive data redacted in logs
- [ ] Configuration logged at startup

**Code Organization:**
- [ ] Focused PRs with no unrelated changes
- [ ] No weasel words in names (`Service`, `Manager`, `Factory`)
- [ ] Document cryptographic algorithm choices
- [ ] No glob re-exports (`pub use foo::*`)

**Safety:**
- [ ] All code is sound — no exceptions
- [ ] `unsafe` only with documented justification
- [ ] Public types implement `Debug` (redacting sensitive data)

**Async:**
- [ ] Futures and public types are `Send`
- [ ] Yield points in long-running CPU-bound tasks
- [ ] Services use `Arc<Inner>` for cheap cloning

**Tooling:**
- [ ] `#[expect]` over `#[allow]` for lint overrides
- [ ] Clippy pedantic + restriction lints enabled
- [ ] cargo-audit, cargo-hack, cargo-udeps in CI
