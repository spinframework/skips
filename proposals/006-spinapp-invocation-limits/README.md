# SKIP 006 - SpinApp invocation limits

Summary: Add support for configuring per-invocation resource limits in `SpinApp` CRD

Owner: Kate Goldenring <kate.goldenring@fermyon.com>

Impacted Projects:

- [x] spin-operator
- [ ] `spin kube` plugin
- [ ] runtime-class-manager
- [ ] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: 05-29-2025

Updated: 05-29-2025

## Background

Spin recently [added support for limiting instance memory](https://github.com/spinframework/spin/pull/3135) and has had an outstanding request for supporting [request timeouts](https://github.com/spinframework/spin/issues/30). The `SpinApp` CRD should support configuring per invocation limits.

## Proposal

Add an `invocationLimits` section to the `SpinApp` CRD with a mapping of limits. Executors can define valid limits that they support and provide a validating webhook for the `SpinApp` resource to avoid unaccepted values being passed.

For example, if the `foo` executor supports  `maxMemory` and `maxExecutionTime` limits, the following `SpinApp` custom resource could be applied:

```yaml
apiVersion: core.spinkube.dev/v1alpha1
kind: SpinApp
metadata:
  name: hello
spec:
  executor: foo
  image: "ghcr.io/spinframework/myapp:latest"
  replicas: 1
  invocationLimits:
    maxMemory: "64Mi"
    maxExecutionTime: "30s"
```

For the Spin containerd shim executor, the Spin Operator can configure the per invocation memory limit by setting the (converted to bytes) value of the `maxMemory` in the `SPIN_MAX_INSTANCE_MEMORY` environment variable on Deployments. This is similar to how OTEL settings [are configured](https://github.com/spinframework/spin-operator/blob/4900cf4bdfdc519c152e7d20f2529fccee7ba40f/internal/controller/deployment.go#L128would).

```yaml
apiVersion: core.spinkube.dev/v1alpha1
kind: SpinApp
metadata:
  name: hello
spec:
  executor: containerd-shim-spin
  image: "ghcr.io/spinframework/myapp:latest"
  replicas: 1
  # Pod level resource limits
  resources:
    limits:
      memory: "128Mi"
  # Per-invocation limits
  invocationLimits:
    maxMemory: "64Mi"
```

## Alternatives Considered

Instead of a generic mapping of limits, these limits could be defined to better validate `SpinApps`; however, a generic map enables executors to diverge in their support of limiting.
