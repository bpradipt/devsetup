# Control Flow and Pattern Matching

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
