# ToDo App Deployment Instructions to Kubernetes

This document provides step-by-step instructions on how to deploy the Django ToDo application to a Kubernetes cluster and explains the architectural choices made for resources, scaling, and update strategies.

## Prerequisites
Before deploying, make sure you have a running Kubernetes cluster (e.g., OrbStack, Minikube, or Kind) and `kubectl` configured.

---

## Deployment Steps

1. **Create the required namespace:**
   ```bash
   kubectl create namespace mateapp
Apply the standalone Pod manifest: (This manifest is used to verify that the Pod spec matches the Deployment spec template).

```bash
kubectl apply -f .infrastructure/pod.yml
Apply the Deployment manifest:
```
```bash
kubectl apply -f .infrastructure/deployment.yml
Apply the Horizontal Pod Autoscaler (HPA):
```
```bash
kubectl apply -f .infrastructure/hpa.yml
Technical Decisions Justification
1. Resource Requests and Limits Choice
Memory (Requests: 64Mi / Limits: 128Mi): A clean Django application in an idle state typically consumes around 35-50Mi of RAM. Setting the request to 64Mi ensures the pod will easily fit on almost any node while guaranteeing enough memory to handle initial user requests. The limit is set to 128Mi to catch potential memory leaks (OOMKilled) without letting a single container drain all the node resources.
CPU (Requests: 100m / Limits: 200m): Django is a synchronous framework, so processing basic views doesn't require substantial CPU power. 100m (0.1 of a core) is optimal for steady operation. The limit of 200m allows temporary spikes during heavy calculations (e.g., handling complex DB queries or API rendering) and provides a clear threshold for the HPA to calculate utilization.
```
```2. Horizontal Pod Autoscaler (HPA) Configuration
Min/Max Pods (2 to 5): The minimum is strictly set to 2 to ensure High Availability (HA). If one pod or node crashes, the second one keeps serving traffic. The maximum of 5 is a sensible upper ceiling to handle traffic spikes (e.g., during testing or presentation verification) while preventing uncontrolled spending of cluster resources.

Triggers (CPU 70% / Memory 80%): * CPU utilization target is set to 70%. CPU spikes are dynamic, so scaling starts before the CPU hits 100% to avoid request queuing and timeouts.

Memory utilization target is set to 80%. Since RAM utilization in Python apps grows more linearly, this threshold safely triggers scaling before the container triggers an OOM-kill event.
```
```3. RollingUpdate Strategy Configuration
maxSurge: 1 / maxUnavailable: 0:

maxUnavailable: 0 guarantees that during an update, zero pods from the old version will be killed until a new pod is fully up, running, and passes readiness probes. This ensures Zero-Downtime.

maxSurge: 1 means Kubernetes will spin up exactly 1 extra pod with the new version during deployment. This requires minimal extra resources on the node (only 64Mi of RAM), which is ideal for small or local clusters.

How to Access the App After Deployment
To access the application locally without creating a complex LoadBalancer or Ingress, use the standard Kubernetes port-forwarding feature:

Find the running pod name (optional, to verify status):
```
```bash
kubectl get pods -n mateapp
Run port-forwarding (maps port 8000 of the deployment to port 8000 on your local machine):
```
```bash
kubectl port-forward deployment/todoapp-deployment 8000:8000 -n mateapp
Open your browser and navigate to: http://localhost:8000
