# Safety and Soundness

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
