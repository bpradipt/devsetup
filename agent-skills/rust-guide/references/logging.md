# Logging and Observability

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
