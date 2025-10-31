# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Cloud API Adaptor is an implementation of the remote hypervisor interface for Kata Containers. It enables creation of Kata Containers VMs ("peer pods") on cloud providers without requiring bare metal worker nodes or nested virtualization support. The project uses Trusted Execution Environments (TEEs) to provide confidential computing capabilities for Kubernetes workloads.

## Repository Structure

This is a **multi-module Go repository** with 6 independent Go modules located in `src/`:

- `src/cloud-api-adaptor` - Main adaptor daemon that implements the remote hypervisor interface
- `src/cloud-providers` - Cloud provider implementations (AWS, Azure, GCP, IBM Cloud, vSphere, libvirt, etc.)
- `src/peerpod-ctrl` - Kubernetes controller for tracking cloud resources and cleanup
- `src/webhook` - Mutating admission webhook for peer pod resource management
- `src/csi-wrapper` - CSI plugins for persistent volume support
- `src/cloud-api-adaptor/podvm` - PodVM image building utilities

Each module has its own `go.mod` and can be built/tested independently.

## Key Architecture Components

### Runtime Components (run on worker nodes)
- **cloud-api-adaptor**: Daemonset that receives commands from `containerd-shim-kata-v2` and manages peer pod VMs via cloud provider APIs. Creates VxLAN tunnels for networking.
- **webhook**: Modifies pod specs to replace resource entries with peer-pod extended resources
- **peerpod-ctrl**: Watches PeerPod events and cleans up dangling cloud resources

### PodVM Components (run inside peer pod VMs)
- **agent-protocol-forwarder**: Bridges network communication between cloud-api-adaptor and kata-agent
- **kata-agent**: Manages containers and workload inside the VM

## Common Commands

### Building

```bash
# Build all binaries (dev build with all providers including libvirt)
cd src/cloud-api-adaptor
make

# Build release binaries (statically linked, excludes libvirt)
RELEASE_BUILD=true make

# Build specific providers only
BUILTIN_CLOUD_PROVIDERS="aws azure" make

# Build specific binary
make cloud-api-adaptor
make agent-protocol-forwarder
```

Note: Dev builds include libvirt provider which requires CGO and libvirt-dev packages (`sudo apt-get install -y libvirt-dev` on Ubuntu). The resulting binary is dynamically linked and OS-specific.

### Testing

```bash
# Unit tests for cloud-api-adaptor
cd src/cloud-api-adaptor
make test

# Unit tests for cloud-providers
cd src/cloud-providers
make test

# E2E tests for specific provider (requires CLOUD_PROVIDER set)
cd src/cloud-api-adaptor
CLOUD_PROVIDER=aws make test-e2e

# Run specific E2E test
CLOUD_PROVIDER=aws RUN_TESTS='TestPodVMCreation' make test-e2e

# Change E2E timeout (default 60m)
TEST_E2E_TIMEOUT=90m CLOUD_PROVIDER=aws make test-e2e
```

### Code Quality

```bash
# Run all checks (from repository root)
make check

# Individual checks
make fmt              # Go formatting
make golangci-lint    # Go linting
make shellcheck       # Shell script linting
make tidy             # Go mod tidy (run on all modules)
make tidy-check       # Verify go.mod is tidy
make govulncheck      # Security vulnerability scanning
make packer-check     # Packer template validation
make terraform-check  # Terraform formatting validation
```

### Building PodVM Images

```bash
cd src/cloud-api-adaptor

# Build podvm-builder image
make podvm-builder

# Build podvm-binaries image
make podvm-binaries

# Build podvm image for specific provider
CLOUD_PROVIDER=aws make podvm-image

# Push to registry (default: quay.io/confidential-containers)
PUSH=true CLOUD_PROVIDER=azure make podvm-image

# Use different registry
REGISTRY=quay.io/bpradipt PUSH=true CLOUD_PROVIDER=azure make podvm-image

# Use different distro (default: ubuntu)
PODVM_DISTRO=rhel make podvm-image
```

### Deployment

```bash
cd src/cloud-api-adaptor

# Deploy cloud-api-adaptor with operator
CLOUD_PROVIDER=aws make deploy

# Deploy without webhook/peerpod-ctrl
CLOUD_PROVIDER=aws RESOURCE_CTRL=false make deploy

# Delete deployment
CLOUD_PROVIDER=aws make delete
```

## Cloud Provider Implementation

Cloud providers are implemented in `src/cloud-providers/<provider>/`:

### Required Files
- `types.go` - Configuration struct with cloud-specific parameters
- `manager.go` - Implements Manager interface (ParseCmd, LoadEnv, NewProvider)
- `provider.go` - Implements Provider interface (CreateInstance, DeleteInstance, Teardown)

### Integration Steps
1. Add provider code in `src/cloud-providers/<provider>/`
2. Create `src/cloud-api-adaptor/cmd/cloud-api-adaptor/<provider>.go` with build tag `//go:build <provider>`
3. Add userdata retrieval support in `src/cloud-api-adaptor/pkg/userdata/provision.go`
4. Update `BUILTIN_CLOUD_PROVIDERS` in Makefile
5. Add E2E tests in `src/cloud-api-adaptor/test/e2e/`

### External Plugins
Providers can be loaded as external plugins at runtime (Go plugin system). This allows adding providers without recompiling. See `src/cloud-api-adaptor/docs/addnewprovider.md` for external plugin implementation details.

## Build Tags

The project uses Go build tags extensively to control which cloud providers are compiled:

- Dev build: `alibabacloud aws azure byom gcp ibmcloud ibmcloud_powervs vsphere libvirt docker`
- Release build: `alibabacloud aws azure gcp ibmcloud ibmcloud_powervs vsphere`

When building with specific providers, set tags: `BUILTIN_CLOUD_PROVIDERS="aws azure" make`

## Testing Strategy

- **Unit tests**: In-tree tests alongside source code (`*_test.go`)
- **E2E tests**: Located in `src/cloud-api-adaptor/test/e2e/`, run against live cloud infrastructure
- **Provider-specific tests**: Use build tags to test individual providers
- Coverage reports: `go test -cover` generates `cover.out`

## Important Environment Variables

### Cloud API Adaptor
- `CLOUD_PROVIDER` - Which provider to use at runtime
- `RELEASE_BUILD` - Set to `true` for statically linked release builds
- `BUILTIN_CLOUD_PROVIDERS` - Space-separated list of providers to compile
- `PODVM_DISTRO` - PodVM base OS (ubuntu, rhel, alinux)
- `REGISTRY` - Container registry for images (default: quay.io/confidential-containers)

### External Plugins
- `ENABLE_CLOUD_PROVIDER_EXTERNAL_PLUGIN` - Enable external plugin loading
- `CLOUD_PROVIDER_EXTERNAL_PLUGIN_PATH` - Path to .so plugin file
- `CLOUD_PROVIDER_EXTERNAL_PLUGIN_HASH` - SHA256 hash of plugin for verification

## Key Package Organization

### src/cloud-api-adaptor/pkg/
- `adaptor/` - Core adaptor server implementation
- `forwarder/` - Agent protocol forwarding
- `podnetwork/` - VxLAN tunnel and network management
- `userdata/` - Cloud-init userdata generation
- `securecomms/` - TLS and secure communication
- `util/` - Common utilities

### src/cloud-providers/
Each provider directory contains:
- `manager.go` - Provider manager implementation
- `provider.go` - VM lifecycle operations
- `types.go` - Provider-specific configuration
- Provider-specific helpers and tests

## Development Workflow

1. Make changes to code
2. Run `make fmt` to format Go code
3. Run `make tidy` to update dependencies
4. Run provider-specific unit tests: `cd src/cloud-providers && make test`
5. Run cloud-api-adaptor tests: `cd src/cloud-api-adaptor && make test`
6. Run linters: `make check` from repository root
7. For provider changes, run E2E tests: `CLOUD_PROVIDER=<provider> make test-e2e`

## Commit Conventions

The project requires Developer Certificate of Origin (DCO 1.1) sign-off for all commits. See `.github/workflows/commit-message-check.yaml` for commit message validation.

## Documentation

Key documentation files:
- `docs/architecture.md` - Overall peer pods architecture
- `src/cloud-api-adaptor/docs/DEVELOPMENT.md` - Development setup
- `src/cloud-api-adaptor/docs/addnewprovider.md` - Adding new cloud providers
- `src/cloud-api-adaptor/docs/SecureComms.md` - Secure communication setup
- `src/cloud-api-adaptor/docs/troubleshooting/` - Troubleshooting guides
