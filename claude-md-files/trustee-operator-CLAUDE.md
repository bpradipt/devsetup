# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

The trustee-operator is a Kubernetes operator that manages the lifecycle of [trustee](https://github.com/confidential-containers/trustee) and its configuration in a Kubernetes cluster. Trustee consists of the Key Broker Service (KBS), Attestation Service (AS), and Reference Value Provider Service (RVPS) for confidential computing workloads.

## Build and Development Commands

### Core Development Workflow

```bash
# Install CRDs into the cluster
make install

# Run the controller locally (foreground, uses current kubeconfig context)
make run

# Build the manager binary
make build

# Run tests with coverage
make test

# Format code
make fmt

# Lint code
make vet

# Generate manifests (CRDs, RBAC, etc.) after API changes
make manifests

# Generate DeepCopy methods after API changes
make generate
```

### Docker Image Build and Deployment

```bash
# Build and push operator image (set REGISTRY env var first)
export REGISTRY=quay.io/youruser
make docker-build docker-push IMG=${REGISTRY}/trustee-operator:latest

# Deploy operator to cluster
make deploy IMG=${REGISTRY}/trustee-operator:latest

# Deploy using prebuilt image
make deploy IMG=quay.io/confidential-containers/trustee-operator:latest

# Undeploy operator
make undeploy

# Uninstall CRDs
make uninstall
```

### End-to-End Integration Tests

```bash
# Run e2e tests in ephemeral kind cluster (requires kuttl and kind installed)
make test-e2e

# Override default images
KBS_IMAGE_NAME=<trustee-image> CLIENT_IMAGE_NAME=<client-image> make test-e2e
```

### Deployment of Sample CRs

```bash
# Navigate to sample directory (microservices is default deployment mode)
cd config/samples/microservices
# or for all-in-one mode:
# cd config/samples/all-in-one

# Generate authentication keys
openssl genpkey -algorithm ed25519 > privateKey
openssl pkey -in privateKey -pubout -out kbs.pem

# Deploy all resources
kubectl apply -k .
```

## Architecture

### Custom Resource: KbsConfig

The operator reconciles a single CRD: `KbsConfig` (api/v1alpha1/kbsconfig_types.go). This CR defines the entire trustee deployment configuration including:

- **Deployment Type**: `AllInOneDeployment` (single container) or `MicroservicesDeployment` (separate KBS, AS, RVPS containers). Default is `MicroservicesDeployment`.
- **ConfigMaps**: References to KBS, AS, RVPS configuration and reference values
- **Secrets**: Authentication keys, HTTPS certificates, client resources
- **Policies**: Attestation and resource policies
- **Platform-specific config**: TDX (Intel Trust Domain Extensions) and IBM SE (Secure Execution) settings
- **Local certificate caching**: For disconnected/air-gapped environments

### Controller Reconciliation Loop

The main reconciliation logic is in `internal/controller/kbsconfig_controller.go`:

1. **Fetch KbsConfig CR**: Retrieves the CR from the cluster
2. **Handle Deletion**: If marked for deletion, runs finalization logic (deletes trustee deployment) and removes finalizer
3. **Deploy/Update Deployment**: Creates or updates the `trustee-deployment` with appropriate containers:
   - `AllInOneDeployment`: Single KBS container with all services
   - `MicroservicesDeployment`: Separate KBS, AS, and RVPS containers
4. **Deploy/Update Service**: Creates/updates `kbs-service` (ClusterIP by default)
5. **Watch Secondary Resources**: Watches ConfigMaps and Secrets referenced in KbsConfig and triggers reconciliation on changes

### Volume Management

The controller constructs complex volume mounts (see `internal/controller/volumes.go`):

- **EmptyDir volumes**: For writable paths like `/opt/confidential-containers` and repository directories
- **ConfigMap volumes**: KBS/AS/RVPS configs, attestation/resource policies, reference values, TDX config
- **Secret volumes**: Auth keys, HTTPS certs, KBS secret resources, local certificate cache
- **PVC volumes**: For IBM SE certificate storage

All volume creation logic is isolated in `volumes.go` for maintainability.

### Key Constants

Constants in `internal/controller/common.go`:

- **Deployment name**: `trustee-deployment`
- **Service name**: `kbs-service`
- **Default namespace**: `trustee-operator-system`
- **Default images**: Can be overridden via env vars (`KBS_IMAGE_NAME`, `AS_IMAGE_NAME`, `RVPS_IMAGE_NAME`, `KBS_IMAGE_NAME_MICROSERVICES`)

### Secondary Resource Watching

The controller watches ConfigMaps and Secrets in its namespace using custom mappers:

- `configMapToKbsConfigMapper`: Triggers reconciliation when any referenced ConfigMap changes
- `secretToKbsConfigMapper`: Triggers reconciliation when any referenced Secret changes

This ensures the deployment is updated when underlying configuration changes.

## Proxy Configuration

The operator is **proxy-aware** and automatically inherits cluster-wide proxy settings on OpenShift clusters.

### Automatic Proxy Detection

When running on OpenShift, the operator automatically reads the cluster-wide proxy configuration (`cluster` Proxy object in `config.openshift.io` API group) and injects the following environment variables into all trustee containers (KBS, AS, RVPS):

- `HTTP_PROXY` / `http_proxy`
- `HTTPS_PROXY` / `https_proxy`
- `NO_PROXY` / `no_proxy`

This happens automatically without any user intervention.

### Manual Proxy Configuration

Users can also manually specify proxy settings via the `KbsEnvVars` field in `KbsConfig`. User-specified values **override** cluster-wide proxy settings:

```yaml
apiVersion: confidentialcontainers.org/v1alpha1
kind: KbsConfig
metadata:
  name: kbsconfig-sample
  namespace: trustee-operator-system
spec:
  KbsEnvVars:
    HTTP_PROXY: "http://custom-proxy.example.com:8080"
    HTTPS_PROXY: "https://custom-proxy.example.com:8080"
    NO_PROXY: "localhost,127.0.0.1,.svc,.cluster.local,custom-domain.com"
```

### Non-OpenShift Clusters

On non-OpenShift Kubernetes clusters, the operator will log a message indicating it cannot retrieve cluster proxy configuration and will only use user-specified `KbsEnvVars` if provided.

## Important Configuration Notes

### Deployment Modes

- **MicroservicesDeployment** (default): KBS, AS, and RVPS run as separate containers in the same pod with localhost communication (AS on port 50004, RVPS on 50003, KBS on 8080)
- **AllInOneDeployment**: All services run in a single KBS container

### Required ConfigMaps/Secrets

For MicroservicesDeployment, you must provide:
- KBS config (kbs-config.toml)
- AS config (as-config.json)
- RVPS config (rvps-config.json)
- RVPS reference values (reference-values.json)
- Auth public key (kbs.pem)

See `config/samples/microservices/` for complete examples.

### Environment Variables

The operator respects these env vars at runtime:
- `POD_NAMESPACE`: Namespace the controller is running in (defaults to `trustee-operator-system`)
- `KBS_IMAGE_NAME`: All-in-one KBS image
- `KBS_IMAGE_NAME_MICROSERVICES`: Microservices KBS image
- `AS_IMAGE_NAME`: Attestation Service image
- `RVPS_IMAGE_NAME`: Reference Value Provider Service image

Users can also inject custom env vars into trustee pods via `KbsEnvVars` in the KbsConfig spec (e.g., `RUST_LOG=debug`).

## Testing Strategy

- **Unit tests**: Use envtest framework with `make test`
- **E2E tests**: Use kuttl in ephemeral kind cluster with sample-attester attestation flow
- The e2e test script creates a kind cluster with local registry and runs attestation workflows

## Platform-Specific Features

### Intel TDX (Trust Domain Extensions)

Configure via `tdxConfigSpec.kbsTdxConfigMapName` to mount `sgx_default_qcnl.conf` into the KBS container.

### IBM Secure Execution

Configure via `ibmSEConfigSpec.certStorePvc` to mount a PVC with IBM SE certificates/keys. See `docs/ibmse.md` for details.

### Intel ITA (Intel Trust Authority)

Special KBS configuration for ITA. See `docs/ita.md` for details.

### Disconnected Environments

Use `kbsLocalCertCacheSpec` to mount local certificates (e.g., VCEK certs for SNP) into the trustee filesystem. See `docs/disconnected.md` for details.

## Key Files

- `api/v1alpha1/kbsconfig_types.go`: KbsConfig CRD definition
- `internal/controller/kbsconfig_controller.go`: Main reconciliation logic
- `internal/controller/volumes.go`: Volume creation and mounting logic
- `internal/controller/common.go`: Constants and helper functions
- `config/samples/microservices/`: Example deployment in microservices mode (default)
- `config/samples/all-in-one/`: Example deployment in all-in-one mode

## Kubebuilder Scaffolding

This operator is built with Kubebuilder. When modifying the API:

1. Edit `api/v1alpha1/kbsconfig_types.go`
2. Run `make manifests` to regenerate CRDs/RBAC
3. Run `make generate` to regenerate DeepCopy methods
4. Update controller logic in `internal/controller/kbsconfig_controller.go`
