# Error Handling

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
