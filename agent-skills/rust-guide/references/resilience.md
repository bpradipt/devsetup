# Resilience

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
