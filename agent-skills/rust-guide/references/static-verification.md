# Static Verification

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
