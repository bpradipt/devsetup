# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains tools and components for Confidential Containers (CoCo), enabling secure container workloads in Trusted Execution Environments (TEEs). The project consists of multiple Rust-based components that work together to provide attestation, encrypted image handling, and confidential data management.

## Architecture

### Main Components

**Attestation Agent (AA)** - Service for TEE attestation procedures
- Provides Key Broker Client (KBC) modules for various Key Broker Services
- Supports multiple TEE platforms via pluggable attesters
- Offers both gRPC and ttRPC service interfaces
- Can be built as a library or standalone process

**Confidential Data Hub (CDH)** - Secure data and key management service
- Orchestrates key management across multiple KMS providers
- Provides REST, gRPC, and ttRPC interfaces
- Handles sealed secrets and resource provisioning
- Integrates with attestation agent for secure key retrieval

**Image Components** - Container image encryption/decryption
- `image-rs`: Container image management library
- `ocicrypt-rs`: OCI image encryption implementation
- `coco-keyprovider`: Key provider for encrypting images

**API Server REST** - Unified REST API gateway
- Exposes CDH and AA services via HTTP endpoints
- Allows containers to access internal services via standard HTTP clients

### Communication Architecture

Components communicate using:
- **ttRPC** (default): Lightweight RPC over Unix sockets for internal communication
- **gRPC**: Alternative RPC for TCP/IP communication
- **REST**: HTTP APIs for external/container access

### Attestation Flow

1. Platform-specific attester generates cryptographic evidence (TDX, SNP, SGX, CCA, SE)
2. Key Broker Client (KBC) communicates with Key Broker Service (KBS)
3. KBS validates attestation evidence
4. Upon successful attestation, keys/secrets are provisioned to the guest

### Feature Flags Architecture

The build system uses feature flags to enable platform-specific code:
- **Attesters**: Platform-specific attestation implementations (tdx-attester, snp-attester, etc.)
- **Resource Providers**: Backend integration (kbs, sev)
- **KMS Providers**: Key management system plugins (aliyun, ehsm)
- **RPC Types**: Protocol selection (grpc, ttrpc)

## Build Commands

### Build All Components

Build for a specific TEE platform:
```shell
make build TEE_PLATFORM=<platform>
```

TEE_PLATFORM options: `none`, `all`, `fs`, `tdx`, `az-tdx-vtpm`, `sev`, `snp`, `amd`, `az-snp-vtpm`, `az-cvm-vtpm`, `se`, `cca`

Build without default resource providers (only offline-fs-kbc):
```shell
make build TEE_PLATFORM=<platform> NO_RESOURCE_PROVIDER=true
```

Install binaries:
```shell
make install DESTDIR=/usr/local/bin
```

### Build Individual Components

**Attestation Agent:**
```shell
cd attestation-agent
make ttrpc=true ATTESTER=<attester> LIBC=<libc>
```
- ATTESTER options: `none`, `all-attesters`, `tdx-attester`, `snp-attester`, `sgx-attester`, `cca-attester`, `se-attester`
- LIBC options: `gnu`, `musl` (note: s390x and ppc64le only support gnu)

Build with gRPC instead of ttRPC:
```shell
make ttrpc=false ATTESTER=<attester>
```

**Confidential Data Hub:**
```shell
cd confidential-data-hub
make RESOURCE_PROVIDER=<providers> KMS_PROVIDER=<providers> RPC=<protocol>
```
- RESOURCE_PROVIDER options: `kbs`, `sev`, `none`, or comma-separated (default: `kbs,sev`)
- KMS_PROVIDER options: `aliyun`, `ehsm`, `none`, or comma-separated (default: `aliyun`)
- RPC options: `ttrpc` (default), `grpc`

Build one-shot binary (runs once and exits):
```shell
make ONE_SHOT=true
```

**API Server REST:**
```shell
cd api-server-rest
make ARCH=<arch> LIBC=<libc>
```

### Architecture-Specific Builds

Cross-compile for different architecture:
```shell
make ARCH=<arch> LIBC=<libc>
```
- ARCH options: `x86_64`, `s390x`, `aarch64`, `powerpc64le`

Build with OpenSSL (required for s390x, aarch64):
```shell
cd attestation-agent
make OPENSSL=1
```

### Debug Builds

Build with debug symbols:
```shell
make DEBUG=1
```

### Update Protocol Buffers

Regenerate ttRPC and gRPC protocol definitions:
```shell
make build-protos
```

## Testing

### Run All Tests for a Component

**Attestation Agent:**
```shell
cd attestation-agent/attestation-agent
cargo test --features openssl,rust-crypto,all-attesters,kbs,coco_as,ttrpc,grpc
```

For specific platforms:
```shell
# s390x
cargo test --no-default-features --features openssl,passport,se-attester,kbs,coco_as

# aarch64/CCA
cargo test --no-default-features --features openssl,rust-crypto,passport,cca-attester,kbs,coco_as,ttrpc,grpc
```

**Confidential Data Hub:**
```shell
cd confidential-data-hub/hub
cargo test --features bin,ttrpc,kbs,sev,aliyun
```

**Run tests for specific packages:**
```shell
cargo test -p attestation-agent -p attester -p kbc -p kbs_protocol
```

### Run Single Test

```shell
cargo test <test_name>
```

### Linting

**Attestation Agent:**
```shell
cd attestation-agent
make lint
```

Platform-specific lint:
```shell
# For s390x
cargo clippy --no-default-features --features openssl,kbs,coco_as,bin,grpc,ttrpc

# For aarch64
cargo clippy --no-default-features --features openssl,rust-crypto,cca-attester,kbs,coco_as,bin,grpc,ttrpc

# Default (x86_64)
cargo clippy --features kbs,coco_as,bin,grpc,ttrpc,all-attesters
```

**Format check:**
```shell
cargo fmt --all -- --check
```

## Development Workflow

### Workspace Structure

This is a Cargo workspace with the following members:
- `api-server-rest`
- `attestation-agent/attestation-agent`
- `attestation-agent/kbc`
- `attestation-agent/kbs_protocol`
- `attestation-agent/attester`
- `attestation-agent/deps/*`
- `attestation-agent/coco_keyprovider`
- `confidential-data-hub/hub`
- `confidential-data-hub/kms`
- `image-rs`
- `ocicrypt-rs`
- `protos`

Dependencies are shared across workspace members via `[workspace.dependencies]` in the root Cargo.toml.

### Adding New TEE Platform Support

1. Create platform-specific attester in `attestation-agent/attester/src/`
2. Add feature flag in component's `Cargo.toml`
3. Update `attestation-agent/Makefile` ATTESTER options
4. Update root `Makefile` TEE_PLATFORM mapping
5. Add CI workflow in `.github/workflows/`

### Binary Variants

Each major component has multiple binary variants:

**Attestation Agent:**
- `ttrpc-aa`: ttRPC protocol (default, Unix socket)
- `grpc-aa`: gRPC protocol (TCP/IP)

**Confidential Data Hub:**
- `ttrpc-cdh`: ttRPC daemon (default)
- `grpc-cdh`: gRPC daemon
- `cdh-oneshot`: One-shot execution mode

### Configuration

**CDH** looks for configuration in this order:
1. Command-line: `confidential-data-hub -c <path>`
2. `/etc/confidential-data-hub.conf`
3. `AA_KBC_PARAMS` environment variable
4. Kernel cmdline: `agent.aa_kbc_params` from `/proc/cmdline`
5. Kata Agent config: `/etc/agent-config.toml` or `KATA_AGENT_CONFIG_PATH`
6. Default: `offline_fs_kbc`

For encrypted image support, set:
```shell
OCICRYPT_KEYPROVIDER_CONFIG=<path-to-ocicrypt_config.json> confidential-data-hub
```

### Client Tools

**CDH ttRPC client:**
```shell
cd confidential-data-hub/hub
cargo build --bin ttrpc-cdh-tool --features bin,ttrpc
```

**CDH gRPC client:**
```shell
cd confidential-data-hub/hub
cargo build --bin grpc-cdh-tool --features bin,grpc
```

## Key Abstractions

### Attestation Agent
- **Attester trait**: Platform-specific attestation evidence generation
- **KBC (Key Broker Client)**: Interface for communicating with KBS
- **Token Provider**: JWT/token generation for authenticated requests

### Confidential Data Hub
- **Resource Provider**: Backend for fetching confidential resources (KBS, SEV pre-attestation)
- **KMS Provider**: Key Management System plugin interface
- **Secret handlers**: Sealed secret unsealing and key operations

## Platform Support Notes

- **s390x/ppc64le**: Must use `LIBC=gnu` (musl not supported), requires OpenSSL
- **aarch64**: OpenSSL support available for compatibility
- **Cross-compilation**: Only tested on Debian-based systems
- **Azure vTPM platforms**: Use `az-tdx-vtpm`, `az-snp-vtpm`, or `az-cvm-vtpm` (which includes both TDX and SNP vTPM attesters) for Azure CVM guests
