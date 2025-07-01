# Kubernetes Operator Pattern

A simple dummy operator that creates and observes desired replica of nginx web-server using kubebuilder
> Please take a look at this [tutorial](https://book.kubebuilder.io/quick-start.html) which includes a quick start to the kubebuilder and how it works

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Architecture](#architecture)
- [Repository Layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Quickstart](#quickstart)
  - [Spin up a local cluster](#spin-up-a-local-cluster)
  - [Build & deploy the operator](#build--deploy-the-operator)
  - [Create a sample resource](#create-a-sample-resource)
  - [Observe reconciliation](#observe-reconciliation)
  - [Uninstall](#uninstall)
- [Configuration](#configuration)
- [Reconciliation Flow](#reconciliation-flow)
- [Status, Conditions & Events](#status-conditions--events)
- [Finalizers & Cleanup](#finalizers--cleanup)
- [Watches, Ownership & Predicates](#watches-ownership--predicates)
- [Webhooks (Optional)](#webhooks-optional)
- [Metrics & Health Probes](#metrics--health-probes)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Security & Production Notes](#security--production-notes)
- [Extending the Operator](#extending-the-operator)
- [FAQ](#faq)
- [License](#license)

---

## Overview

**Kubernetes Operator Pattern** encapsulates domain-specific automation (day-2 operations) inside a controller that watches Custom Resources (CRs) and reconciles actual cluster state toward a desired spec. This repository is intended as a hands-on guide and scaffold for building your own Operator. It follows common best practices you’ll see in Kubebuilder/operator-sdk projects (controller-runtime, Envtest, make targets, etc.).

---

## Learning Objectives

By the end, you should be able to:

- Scaffold a **CustomResourceDefinition (CRD)** and a **controller**.
- Implement a **Reconcile** loop that creates/updates dependent resources.
- Track **Status** and **Conditions** to surface progress and errors.
- Emit **Events** and meaningful logs for observability.
- Use **Finalizers** to perform cleanup on deletion.
- Configure **watches** and **ownership** so garbage collection works as expected.
- Add **webhooks** (mutating/validating) to enforce policy and defaults.
- Package the operator into a container image and run it in a cluster.
- Write **tests** using Envtest and controller-runtime fakes.

---

## Architecture

At a high level:

1. **CRD** defines your API (`spec` = desired state, `status` = observed state).
2. **Controller** watches CRs and related resources (e.g., Deployments, Services).
3. The **Reconcile** function computes diffs and applies changes via the API server.
4. **Status/Conditions** are updated to inform users and CI/CD systems.
5. **Finalizers** ensure off-cluster or persistent resources are cleaned up.
6. Optional **webhooks** validate or default incoming objects.

---

## Repository Layout

If you used Kubebuilder-style scaffolding, you’ll typically see something like:

```
.
├── api/                     # CRD Go types (Spec/Status), +deepcopy
├── config/
│   ├── crd/                 # Generated CRDs
│   ├── default/             # Kustomize base for default install
│   ├── manager/             # Manager (deployment) manifests
│   ├── rbac/                # Roles/Bindings/ServiceAccount
│   ├── samples/             # Sample CR instances (YAML)
│   └── webhook/             # (Optional) webhook configuration
├── controllers/             # Reconciler implementations
├── hack/                    # Helper scripts
├── internal/                # (Optional) internal packages
├── main.go                  # Manager entrypoint
├── Makefile                 # Build/test/deploy targets
├── go.mod / go.sum
└── README.md
```

> Your repository might use a different layout or add extra modules; the commands below still apply with minor path/name changes.

---

## Prerequisites

- **Go** ≥ 1.20
- **Docker** (or podman) to build images
- **kubectl** (v1.24+ recommended)
- A local Kubernetes cluster (one of):
  - **kind**, **minikube**, or **k3d**
- (Recommended) **make** and **kustomize**

---

## Quickstart

### Spin up a local cluster

Using **kind**:

```bash
kind create cluster --name operator-pattern
```

> Prefer minikube? Use `minikube start` and replace the `kind`-specific commands below with minikube equivalents.

### Build & deploy the operator

Build the container image and load it into your cluster:

```bash
# Build image
make docker-build IMG=operator-pattern:dev
# Load image into kind (skip for minikube with registry access)
kind load docker-image operator-pattern:dev --name operator-pattern

# Deploy RBAC, CRDs, manager using kustomize overlays
make deploy IMG=operator-pattern:dev
```

> If your project does not provide a `Makefile`, use `docker build -t operator-pattern:dev .` and `kubectl apply -k config/default`.

### Create a sample resource

Apply a sample CR from `config/samples/` (adjust group/version/kind and file name to your repo):

```bash
kubectl apply -f config/samples/<group>_<version>_<kind>.yaml
```

Example skeleton if you need one:

```yaml
apiVersion: apps.example.com/v1alpha1
kind: Sample
metadata:
  name: sample-demo
spec:
  replicas: 2
  image: ghcr.io/your/image:tag
  config:
    message: "Hello from the Operator Pattern!"
```

### Observe reconciliation

Check controller logs and created resources:

```bash
# Manager logs
kubectl logs -n <manager-namespace> deploy/<manager-name> -f

# View the CR
kubectl get sample sample-demo -o yaml

# List owned resources
kubectl get all -l app.kubernetes.io/managed-by=operator-pattern
```

> Use a consistent set of labels (e.g., `app.kubernetes.io/*`) in your owned objects to make discovery and cleanup easier.

### Uninstall

Remove the sample CR and undeploy the operator:

```bash
kubectl delete -f config/samples/<group>_<version>_<kind>.yaml --ignore-not-found
make undeploy
kind delete cluster --name operator-pattern
```

---

## Configuration

Common runtime configuration knobs (via environment variables or flags on the manager):

| Setting / Env Var            | Purpose                                   | Example |
|-----------------------------|-------------------------------------------|---------|
| `WATCH_NAMESPACE`           | Namespace scoping (blank = cluster-wide)  | `apps`  |
| `LEADER_ELECT`              | Enable leader election for HA             | `true`  |
| `METRICS_ADDR`              | Metrics endpoint                           | `:8080` |
| `HEALTH_PROBE_ADDR`         | Health/ready probes                        | `:8081` |
| `LOG_LEVEL`                 | Logging verbosity                          | `info` / `debug` |

> If you’re using the standard controller-runtime manager, these are often flags on the binary or env vars in the Deployment manifest.

---

## Reconciliation Flow

A typical reconcile implements the following stages:

1. **Fetch** the CR instance; if not found, return (object deleted).
2. **Initialize** defaults and ensure the finalizer is present.
3. **Observe**: query child resources (e.g., Deployments/Services/ConfigMaps).
4. **Decide**: compute desired vs. actual state.
5. **Act**: create/update/delete children to converge on desired state.
6. **Report**: update `.status` and emit **Events**.
7. **Requeue** when needed (rate-limited) or rely on watches for changes.

Pseudocode:

```go
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    obj := &v1alpha1.Sample{}
    if err := r.Get(ctx, req.NamespacedName, obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 1) Handle deletion with finalizer
    if !obj.ObjectMeta.DeletionTimestamp.IsZero() {
        return r.finalize(ctx, obj)
    }
    if addFinalizer(obj) { defer r.Update(ctx, obj) }

    // 2) Observe current children
    deploy := &appsv1.Deployment{}
    // ... fetch/create/update as needed

    // 3) Compute desired state & patch
    desired := buildDesiredDeployment(obj)
    if err := controllerutil.SetControllerReference(obj, desired, r.Scheme); err != nil { /* ... */ }
    if err := r.apply(ctx, desired); err != nil { /* ... */ }

    // 4) Update status & conditions
    updateStatus(obj, deploy)

    return ctrl.Result{RequeueAfter: time.Minute}, nil
}
```

---

## Status, Conditions & Events

Expose progress and errors through `.status` and **Conditions** so users and automation can reason about state:

```yaml
status:
  observedGeneration: 3
  readyReplicas: 2
  conditions:
  - type: Ready
    status: "True"
    reason: ComponentsHealthy
    message: "Deployment available with 2/2 ready replicas"
    lastTransitionTime: "2025-10-31T22:00:00Z"
```

Emit **Events** for user-visible feedback (e.g., `Normal Created`, `Warning UpdateFailed`).

---

## Finalizers & Cleanup

When a CR is deleted, the API server sets a **deletion timestamp**. Your controller should:

1. Detect deletion.
2. Run cleanup (e.g., remove external resources, S3 buckets, DNS records).
3. Remove the finalizer and update the object so Kubernetes can complete deletion.

This prevents **orphaned** external resources.

---

## Watches, Ownership & Predicates

- Use `Owns()` so the controller receives reconcile events when child objects change.
- Set **owner references** on children for automatic garbage collection.
- Add **predicates** (e.g., only reconcile on generation changes) to reduce churn.
- Consider secondary watches (e.g., watch `Secrets` referenced in your CR `spec`).

---

## Webhooks (Optional)

Add **Mutating** webhooks for defaults and **Validating** webhooks for policy (e.g., schema-level constraints that go beyond OpenAPI). Typical steps:

- Implement `webhook.Defaulter` (`Default()` method) and/or `webhook.Validator` (`ValidateCreate/Update/Delete`).
- Enable webhook manager and TLS certs in `config/webhook/`.
- Deploy `ValidatingWebhookConfiguration` / `MutatingWebhookConfiguration` with the manager Service.

> During local development, you can run `make run` and use `kubebuilder`’s `webhook-cert` helpers or cert-manager for TLS in-cluster.

---

## Metrics & Health Probes

The manager exposes:

- **Metrics** on `/metrics` (Prometheus format, default `:8080`).
- **Liveness** at `/healthz` and **readiness** at `/readyz` (default `:8081`).

Wire these into your cluster’s monitoring and set PodDisruptionBudgets / resource requests for reliability.

---

## Testing

- **Unit tests** against pure logic.
- **Envtest** for API-server-level tests without a real cluster:

```bash
make test
```

Envtest spins up a local control plane to exercise reconcilers against real CRDs in isolated tests.

---

## Troubleshooting

- **Controller not reconciling**  
  - Check RBAC: the ServiceAccount must have `get/list/watch` on your CRD and children.
  - Verify watches/ownership are set; children without owner refs won’t trigger events.

- **Status never updates**  
  - Ensure you call `Status().Update` and that the Role includes `update` on `status` subresource.

- **Webhooks time out**  
  - Confirm Service name/port and CA bundle in the webhook configuration.
  - Ensure the manager Pod is `Ready` and serving TLS.

- **Image pull errors in kind**  
  - Run `kind load docker-image operator-pattern:dev --name operator-pattern` after each rebuild.

- **CR stuck in Deleting**  
  - Check finalizer logic; log and handle errors during cleanup, then remove the finalizer.

---

## Security & Production Notes

- Use dedicated **ServiceAccounts** and least-privilege **RBAC**.
- Enable **leader election** for HA if running replicas > 1.
- Treat any external credentials (cloud, SMTP, DB) as **Secrets** and avoid logging them.
- Set **resource requests/limits**, **probes**, and **Pod Security** settings appropriate to your environment.
- Pin dependencies and scan images; publish SBOMs if required.

---

## Extending the Operator

Ideas to extend this repository:

- Add a second controller for a related resource (e.g., sidecar config).
- Implement **horizontal scaling** or **rollout strategies** driven by CR `spec`.
- Support **backup/restore** or **snapshot** flows.
- Add **Grafana dashboards** for metrics and condition summaries.
- Package a **Helm chart** for installation.

---

## FAQ

**Q: Cluster-scoped or namespaced CRD?**  
A: Prefer **namespaced** unless you explicitly manage cluster-wide state.

**Q: Should I use webhooks for validation instead of OpenAPI schema?**  
A: Use **OpenAPI** for structural constraints; add **validating webhooks** for cross-field or external checks.

**Q: Can I watch external resources (e.g., cloud APIs)?**  
A: Yes, but design for **idempotency**, **timeouts**, and **rate-limits**; reflect external state in `.status`.

---

