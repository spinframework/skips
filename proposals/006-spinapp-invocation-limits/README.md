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

Add an `invocationLimits` section to the `SpinApp` CRD with a mapping of limits. The limits can be of two types: common and executor namespaced. The project will define common keys that executors *should* support, starting with `memory`. Executors can define additional limits that they support as namespaced keys; for example, the key for a `bar` limit specific to the Spintainer executor may be `spintainer.spinframework.dev/bar`.

The Spin Operator will validate common limits. For `memory`, it will assert that the requested value does not exceed that of the `resources.limits.memory` section. Executors are responsible for documenting and validating their limits. One approach is to provide a validating webhook for the `SpinApp` resource to avoid unaccepted values being passed.

The following is an example `SpinApp` that uses the `foo` executor, which supports the common `memory` limit and an executor specific `foo.dev/executionWallTime` limit:

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
    memory: "64Mi"  
    foo.dev/executionWallTime: "30s"  
```

For the Spin containerd shim executor, the Spin Operator can configure the per invocation memory limit by setting the (converted to bytes) value of the `memory` in the `SPIN_MAX_INSTANCE_MEMORY` environment variable on Deployments. This is similar to how OTEL settings [are configured](https://github.com/spinframework/spin-operator/blob/4900cf4bdfdc519c152e7d20f2529fccee7ba40f/internal/controller/deployment.go#L128).

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
    memory: "64Mi"
```

## Alternatives Considered

Instead of a generic mapping of limits, these limits could be defined to better validate `SpinApps`; however, a generic map enables executors to diverge in their support of limiting.

Instead of adding an `invocationLimits` section, the `resources.limits` section could be overloaded with invocation limit specific resources. While this is an option, not all invocation limits map to the concept of a "resource", i.e. `executionWallTime`. Also, the `resources` must be Kubernetes resources, so these limits syntactically would be [Node extended resources](https://kubernetes.io/docs/tasks/administer-cluster/extended-resource-node/). There is concern that SpinApp overloading these concepts could lead to confusion. Finally, adding an distinct `invocationLimits` section highlights a key advantage of SpinKube: support of per invocation limits.
