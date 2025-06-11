# SKIP 000 -Sidecar Containers in Spin Operator

Summary: This proposal introduces sidecar container support in the Spin Operator, enabling Spin applications to run alongside Linux containers within the same pod. This allows for added functionality such as monitoring, logging, and security.

Owner: jiazh@microsoft.com

Impacted Projects:

- [x] spin-operator
- [ ] `spin kube` plugin
- [ ] runtime-class-manager
- [ ] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: October 16th, 2024

Updated: October 16th, 2024

## Background

The [Runwasi] project supports running Wasm workloads in the same pod as Linux containers. Currently, the SpinApp Custom Resource Definition (CRD) does not support running [Sidecar Containers]. Sidecars are important in use cases where additional services need to run in the same pod as the Spin application, enabling low-latency communication and resource sharing.

[Sidecar Containers]: (https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
[Runwasi]: (https://github.com/containerd/runwasi)
[SpinApp CRD]: (https://www.spinkube.dev/docs/reference/spin-app/)
[Spin Operators repository]: (https://github.com/spinkube/spin-operator)

### User Stories

- As an user, I have a proprietary solution packaged as a binary that cannot be ported to Wasm. I would like to run this binary as a Linux container. Also, I would like to run this container in the same pod as the Spin application so that they can communicate with each other locally. In addition, I want the sidecar container to start before the Spin application.
- As a developer, I would like to run [Dapr] sidecar containers in the same pod as the Spin application so that I can leverage Dapr's capabilities, such as service invocation, state management, pub/sub, etc, in my Spin application.
- As a developer, I would like to run a [Envoy] proxy sidecar container in the same pod as the Spin application so that I can re-use the service mesh capabilities provided by Envoy, such as traffic management, security, observability, etc, in my Spin application.

[Dapr]: (https://dapr.io/)
[Envoy]: (https://www.envoyproxy.io/)

### Existing applications

Here are some existing prototypes that I have found that runs Wasm workloads side by side with Linux containers:

- [SpinKube - the first look at WebAssembly/WASI application (SpinApp) on Kubernetes](https://dev.to/thangchung/spinkube-the-first-look-at-webassemblywasi-application-spinapp-on-kubernetes-36jd) and [Coffeeshop repo](https://github.com/thangchung/coffeeshop-on-spinkube/tree/feat/global-azure-2024)
- [Istio Wasm Demo](https://github.com/keithmattix/istio-wasm-demo)


## Proposal


The main changes are

1. A new field `InitContainers` in the `spinapp_types.go` that defines a list of sidecar containers to be included in the Deployment.
```go
// InitContainers defines the list of sidecar containers to be included in the deployment.
//
// These containers will not include the main Spin App. They share the Spin App's
// environment variables and volumes.
// +kubebuilder:validation:Optional
InitContainers []corev1.Container `json:"containers,omitempty"`
```

2. Adds containers to the deployment creation in `spinapp_controller.go`:
```go
var containers []corev1.Container
if len(app.Spec.InitContainers) > 0 {
    for _, c := range app.Spec.InitContainers {
        if c.Image == app.Spec.Image {
            return nil, errors.New("container in app.Spec.InitContainers must have a different image than Spin App")
        }
        if c.Name == "" {
            return nil, errors.New("container in app.Spec.InitContainers must have a name")
        }
        if c.Name == app.Name {
            return nil, errors.New("container in app.Spec.InitContainers must have a different name than the Spin App")
        }
        c.Env = append(c.Env, env...)
        c.VolumeMounts = append(c.VolumeMounts, volumeMounts...)
        if c.Resources.Limits == nil && c.Resources.Requests == nil {
            c.Resources = resources
        }
        if c.LivenessProbe == nil {
            c.LivenessProbe = livenessProbe
        }
        if c.ReadinessProbe == nil {
            c.ReadinessProbe = readinessProbe
        }
        containers = append(containers, c)
    }
}
```

See the full diff [here](https://github.com/spinkube/spin-operator/compare/main...Mossaka:spin-operator-msk:sidecars)

[sidecar containers]: (https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
[InitContainers]: (https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)

### Ports

Sidecar containers must avoid using port 80, which is reserved for the Spin application. Other ports like 8080 or 8081 can be used.

### Services

Sidecar containers can communicate internally using `localhost`, but if external access is needed, users must create a separate service for the sidecar container.

### Restrictions

* Sidecar containers cannot use the same image or name as the Spin App. 
* Sidecar containers must not use port 80.

### Sample SpinApp yaml

```yaml
apiVersion: core.spinoperator.dev/v1alpha1
kind: SpinApp
metadata:
  name: simple-spinapp
spec:
  image: "ttl.sh/spinapp:44h"
  replicas: 1
  executor: containerd-shim-spin
  volumes:
    - name: example-volume
      persistentVolumeClaim:
        claimName: example-pv-claim
  volumeMounts:
    - name: example-volume
      mountPath: "/mnt/data"
  initContainers:
    - name: logshipper
      image: alpine:latest
      restartPolicy: Always
      command: ['sh', '-c', 'tail -F /opt/logs.txt']
      volumeMounts:
      - name: example-volume
        mountPath: "/mnt/data"
```

Note: The `InitContainers` field is intentionally designed to follow the the design of sidecar containers in Kubernetes Pods. You will need to have `SidecarContainers` feature gate enabled in your Kubernetes cluster to use this feature. See more details [here](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/).

### Init Containers

Since the sidecar containers are just a special case of InitContainers, I realized that the scope of this SKIP is naturally expanded to include the support of init containers in the Spin Operator. 

Examples of init containers include:

```yaml
apiVersion: core.spinoperator.dev/v1alpha1
kind: SpinApp
metadata:
  name: simple-spinapp
spec:
  image: "ttl.sh/spinapp:44h"
  replicas: 1
  executor: containerd-shim-spin
  volumes:
    - name: example-volume
      persistentVolumeClaim:
        claimName: example-pv-claim
  volumeMounts:
    - name: example-volume
      mountPath: "/mnt/data"
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

## Alternatives Considered

1. Use `Containers` instead of `InitContainers` in the SpinApp CRD.
2. Use `SidecarContainers` instead of `InitContainers` in the SpinApp CRD and automatically set the `RestartPolicy` to `Always` in the Deployment.
3. Automatically setting up services for sidecar containers in the operator.

---

This proposal adds sidecar container support in Spin Operator, addressing key use cases for running sidecar containers alongside Spin applications.

