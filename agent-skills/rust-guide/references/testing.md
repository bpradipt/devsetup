# Testing

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
