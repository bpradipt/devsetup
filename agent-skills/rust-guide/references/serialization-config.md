# Serialization and Configuration

### 5.1 Use `#[serde(default)]` deliberately

Only add `#[serde(default)]` when a field genuinely has a meaningful default. Don't add it just to make tests pass — it can mask missing configuration.

```rust
pub struct IntelTrustAuthorityConfig {
    pub api_key: String,
    pub certs_file: String,
    pub allow_unmatched_policy: Option<bool>,
    #[serde(default)]  // Only if this field is truly optional
    pub policy_ids: Vec<String>,
}
```

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)

### 5.2 Use `Default` trait with `impl Default` for config structs

When removing explicit defaults from config builder code, make sure `impl Default` provides the same values.

> **[trustee]** [PR #776](https://github.com/confidential-containers/trustee/pull/776)
