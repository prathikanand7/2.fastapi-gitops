## 3.4 Deployment on a Kubernetes "Production" Cluster

The FastAPI application was deployed on a local Kubernetes cluster using Minikube
and Helm. Minikube was started with the ingress addon enabled and the Minikube IP
was added to `/etc/hosts` so that `minikube.test` resolved correctly. The
application was installed using the provided Helm chart with a custom values file:

```bash
helm install my-release ./helm/fastapi-gitops-starter -f custom-values.yaml
```

This created a Deployment, a Service, an Ingress resource, and a Horizontal Pod
Autoscaler. The application was accessible at:
`http://minikube.test/GitOps-Starter/api/items`

A load test was run using `hey` for 30 seconds:

```bash
hey -z 30s http://minikube.test/GitOps-Starter/api/items
```

The HPA detected that CPU utilisation exceeded the 10% threshold and started
scaling up. New pods were visible transitioning through Pending, ContainerCreating,
and Running states using `kubectl get pods -w`. Replicas increased from 1 to
several pods under load, then scaled back down once the test finished.

## 3.5 Questions

### 1. The auto-scaling did not work as expected. What could be the possible reasons?

HPA may fail for several reasons. The most common is that `resources.requests`
for CPU or memory are not set in the Deployment manifest. Without a base request
value, the HPA cannot compute a utilisation percentage and reports `<unknown>`.
The metrics-server also needs to be running (`minikube start --addons=metrics-server`).
Other causes include the target threshold being set too high relative to actual
load, or the built-in stabilisation windows (5 minutes scale-down, 15 seconds
scale-up) making it appear as if nothing is happening.

### 2. How does Horizontal Pod Autoscaling (HPA) work in Kubernetes?

HPA runs as a control loop in the Kubernetes controller manager. Every 15 seconds
it queries the metrics API for current resource utilisation and computes:

desiredReplicas = ceil(currentReplicas x (currentMetric / desiredMetric))

If the result differs from the current count, HPA updates `spec.replicas` on the
Deployment. Scale-up is fast (seconds), scale-down is deliberately slow to avoid
thrashing. HPA can operate on CPU, memory, or custom metrics via the custom
metrics API.
