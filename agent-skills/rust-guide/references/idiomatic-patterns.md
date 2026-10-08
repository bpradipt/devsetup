# Idiomatic Patterns

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
