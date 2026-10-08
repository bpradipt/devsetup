---
name: rust-guide
description: Rust coding guidelines from confidential-containers/trustee PR reviews and Microsoft Pragmatic Rust Guidelines. Use when writing, reviewing, or refactoring Rust code — error handling, control flow, API design, testing, logging, unsafe/soundness, async, clippy/lint setup.
---

# Rust Coding Guidelines

Guidelines merged from [trustee](https://github.com/confidential-containers/trustee) PR review comments and [Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/).

Sources are tagged: **[trustee]** for project-specific PR feedback, **[MS]** for Microsoft guidelines.

## How to use

1. Apply the checklist below to all Rust code you write or review.
2. For Do/Don't examples and rationale, read the reference file for the topic you are working on.

| Topic | Detail |
|-------|--------|
| [Error Handling](references/error-handling.md) | 6 rules |
| [Control Flow and Pattern Matching](references/control-flow.md) | 5 rules |
| [API and Function Design](references/api-design.md) | 8 rules |
| [Testing](references/testing.md) | 4 rules |
| [Serialization and Configuration](references/serialization-config.md) | 2 rules |
| [Logging and Observability](references/logging.md) | 5 rules |
| [Code Organization](references/code-organization.md) | 7 rules |
| [Idiomatic Patterns](references/idiomatic-patterns.md) | 5 rules |
| [Safety and Soundness](references/safety.md) | 2 rules |
| [Async and Concurrency](references/async-concurrency.md) | 3 rules |
| [Static Verification](references/static-verification.md) | 3 rules |
| [Resilience](references/resilience.md) | 2 rules |

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
