# Code Organization

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
