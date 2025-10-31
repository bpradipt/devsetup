# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the **Trustee** repository for Confidential Containers, containing tools and components for attesting confidential guests and providing secrets to them. The repository implements a RATS (Remote ATtestation procedureS) architecture with three main components:

- **Key Broker Service (KBS)**: Acts as a Relying Party, facilitating remote attestation and secret delivery
- **Attestation Service (AS/CoCo-AS)**: Acts as a Verifier, verifying TEE (Trusted Execution Environment) evidence
- **Reference Value Provider Service (RVPS)**: Manages reference values used to verify TEE evidence

## Repository Structure

This is a Cargo workspace with the following main components:

```
trustee/
├── kbs/                      # Key Broker Service
├── attestation-service/      # Attestation Service (AS)
├── rvps/                     # Reference Value Provider Service
├── tools/
│   ├── kbs-client/          # KBS client SDK and CLI tool
│   └── trustee-cli/         # Unified CLI for Trustee operations
├── deps/
│   ├── verifier/            # Platform-specific TEE verifiers (TDX, SGX, SNP, CCA, CSV, SE, NVIDIA)
│   └── eventlog/            # Event log handling
└── integration-tests/        # Integration tests
```

## Build Commands

### KBS (Key Broker Service)

Build KBS in background check mode (most common):
```bash
cd kbs
make background-check-kbs
# With specific features:
make background-check-kbs AS_TYPE=coco-as COCO_AS_INTEGRATION_TYPE=grpc ALIYUN=true
```

Build KBS in passport mode (requires two KBS instances):
```bash
cd kbs
make passport-issuer-kbs      # KBS for attestation
make passport-resource-kbs    # KBS for resource provisioning
```

Build KBS client:
```bash
cd kbs
make cli
# For static Linux binary:
make cli-static-linux
```

### Attestation Service

Build AS binaries (both gRPC and RESTful):
```bash
cd attestation-service
make build
# Or build individually:
make grpc-as
make restful-as
```

Build with specific verifier:
```bash
make build VERIFIER=tdx-verifier    # Intel TDX only
make build VERIFIER=all-verifier    # All verifiers (default)
```

Enable RVPS gRPC mode:
```bash
make build RVPS_GRPC=true
```

### RVPS (Reference Value Provider Service)

```bash
cd rvps
make build     # Builds both rvps binary and rvps-tool
make bin       # Build only rvps service
make tool      # Build only rvps-tool client
```

### Trustee CLI

```bash
cd tools/trustee-cli
make build
```

### Workspace-wide

Build all components from repository root:
```bash
cargo build --release --workspace
```

## Testing

### KBS Tests

```bash
cd kbs
make check                                    # Run all KBS tests
make check TEST_FEATURES="sample_only"        # Test with specific features
```

### Attestation Service Tests

```bash
cd attestation-service
cargo test --features grpc-bin,all-verifier,rvps-grpc
```

### Integration Tests

Integration tests are in `integration-tests/` and test the full stack:
```bash
cargo test -p integration-tests
```

### OPA Policy Checking (AS)

The AS uses OPA (Open Policy Agent) for policy evaluation:
```bash
cd attestation-service
make opa-check    # Validates all .rego policy files
```

## Linting and Formatting

### KBS
```bash
cd kbs
make lint      # Run clippy
make format    # Check code formatting
```

### Attestation Service
```bash
cd attestation-service
cargo clippy --features grpc-bin,all-verifier -- -D warnings
cargo fmt -- --check --config format_code_in_doc_comments=true
```

### Trustee CLI
```bash
cd tools/trustee-cli
make lint
make format
```

## Running Locally

### Quick Start with Docker Compose

The easiest way to run the full Trustee stack locally:

```bash
# Generate auth keys (first time only)
openssl genpkey -algorithm ed25519 > kbs/config/private.key
openssl pkey -in kbs/config/private.key -pubout -out kbs/config/public.pub

# Start the cluster (KBS + AS + RVPS + keyprovider)
docker compose up -d
```

This starts:
- KBS on port 8080
- AS (gRPC) on port 50004
- RVPS on port 50003
- CoCo Keyprovider on port 50000

### Running Individual Components

Run KBS:
```bash
cd kbs
cargo run --release --no-default-features --features coco-as-builtin -- \
  --config-file config/kbs-config.toml
```

Run AS (gRPC):
```bash
cd attestation-service
cargo run --bin grpc-as --features grpc-bin,all-verifier,rvps-grpc -- \
  --socket 127.0.0.1:50004 \
  --config-file config/as-config.json
```

Run AS (RESTful):
```bash
cd attestation-service
cargo run --bin restful-as --features restful-bin,all-verifier,rvps-grpc -- \
  --config-file config/as-config.json
```

Run RVPS:
```bash
cd rvps
cargo run --release --no-default-features --features bin -- \
  --address 127.0.0.1:50003
```

## Architecture Details

### KBS Integration Modes

**Background Check Mode** (most common):
- KBS connects directly to AS for evidence verification
- AS can be integrated as builtin crate (`COCO_AS_INTEGRATION_TYPE=builtin`) or remote gRPC service (`COCO_AS_INTEGRATION_TYPE=grpc`)
- Single KBS handles both attestation and resource provisioning

**Passport Mode**:
- Two KBS instances: one for attestation (issuer-kbs), one for resources (resource-kbs)
- Decouples attestation from resource provisioning
- Useful when these functions are handled by separate entities

### AS Verifier Drivers

The AS supports multiple TEE platforms through verifier drivers in `deps/verifier/src/`:
- `tdx/` - Intel Trust Domain Extensions
- `sgx/` - Intel Software Guard Extensions
- `snp/` - AMD SEV-SNP
- `cca/` - ARM Confidential Compute Architecture
- `csv/` - Hygon China Security Virtualization
- `se/` - IBM Secure Execution
- `az_snp_vtpm/` - Azure SNP with vTPM
- `az_tdx_vtpm/` - Azure TDX with vTPM
- `nvidia/` - NVIDIA devices
- `sample/` - Dummy verifier for testing

### RVPS Integration

RVPS can be integrated with AS in two modes:
- **Native Mode**: RVPS as a crate inside AS binary (not recommended)
- **gRPC Mode**: AS connects to remote RVPS service (build AS with `RVPS_FEATURES=rvps-grpc`)

### Policy Engine

The AS uses OPA (regorus engine) for policy evaluation. Policy files are `.rego` files located in:
- `attestation-service/src/ear_token/*.rego`

Default policy: `ear_default_policy_cpu.rego`

## Cross-Compilation

The Makefiles support cross-compilation for x86_64, s390x, and aarch64:

```bash
cd kbs
make background-check-kbs ARCH=aarch64
```

Requirements for cross-compilation (Debian-based systems):
- Automatically installs cross-compiler toolchain
- Sets appropriate OpenSSL library paths
- Only tested on Debian-like OSes

## Configuration

### KBS Configuration
- Config format: TOML
- Location: `kbs/config/kbs-config.toml`
- Documentation: `kbs/docs/config.md`

### AS Configuration
- Config format: JSON
- Location: `kbs/config/as-config.json`
- Documentation: `attestation-service/docs/config.md`

### RVPS Configuration
- Config format: JSON
- Supports storage backends: LocalFs, LocalJson
- Example in `rvps/README.md`

## Key Files and Locations

- **Resource storage**: `kbs/data/kbs-storage/` (when using LocalFs backend)
- **Reference values**: `kbs/data/reference-values/` (RVPS storage)
- **Auth keys**: `kbs/config/private.key` and `kbs/config/public.pub`
- **Attestation token signing**: `kbs/config/token.key` and `kbs/config/token-cert.pem`

## Important Notes

- The repository was formerly named `kbs` but is now `trustee` (references to `kbs` in code paths are historical)
- When working with policies, always validate `.rego` files using `make opa-check`
- The AS outputs EAR (EAT Attestation Results) tokens in JWT format
- Integration tests are architecture-specific (different verifiers for x86_64, s390x, aarch64)
- Default config blocks sample evidence in production for security
