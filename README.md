# Voting App on Kubernetes

A multi-service voting application deployed on Kubernetes using plain Pod and Service manifests (`kubectl` only, no Helm).

Users vote between two options on the **vote** page, and the results appear live on the **result** page.

## Architecture

| Component | Image | Role | Service type | Port |
|-----------|-------|------|--------------|------|
| `vote` | `dockersamples/examplevotingapp_vote` | Voting web UI | NodePort | 8080 → 80 (nodePort `31000`) |
| `redis` | `redis:alpine` | Temporary store for incoming votes | ClusterIP | 6379 |
| `worker` | `dockersamples/examplevotingapp_worker` | Moves votes from Redis to Postgres | None | n/a |
| `db` | `postgres:15-alpine` | Permanent vote storage | ClusterIP | 5432 |
| `result` | `dockersamples/examplevotingapp_result` | Results web UI | NodePort | 8081 → 80 (nodePort `31001`) |

### Data flow

```
vote  ──►  redis  ──►  worker  ──►  db  ──►  result
(UI)      (queue)    (processor)  (storage)   (UI)
```

## Repository Structure

```
.
├── voting-app-pod.yaml      # vote Pod
├── voting-service.yaml      # vote Service (NodePort 31000)
├── redis-pod.yaml           # redis Pod
├── redis-service.yaml       # redis Service (ClusterIP)
├── worker-pod.yaml          # worker Pod
├── db-pod.yaml              # postgres Pod
├── db-service.yaml          # db Service (ClusterIP)
├── result-app-pod.yaml      # result Pod
└── result-service.yaml      # result Service (NodePort 31001)
```

## Prerequisites

- A running Kubernetes cluster (Minikube, kind, Docker Desktop, or any other)
- `kubectl` installed and configured to talk to the cluster

Check your setup:

```bash
kubectl cluster-info
kubectl get nodes
```

## Deployment

Clone the repository:

```bash
git clone https://github.com/nikhil-flux/voting-app-k8s.git
cd voting-app-k8s
```

Deploy everything:

```bash
kubectl apply -f .
```

Or deploy component by component:

```bash
# Data layer
kubectl apply -f redis-pod.yaml -f redis-service.yaml
kubectl apply -f db-pod.yaml -f db-service.yaml

# Application layer
kubectl apply -f worker-pod.yaml
kubectl apply -f voting-app-pod.yaml -f voting-service.yaml
kubectl apply -f result-app-pod.yaml -f result-service.yaml
```

## Verify

```bash
kubectl get pods
kubectl get services
```

All 5 pods (`vote`, `redis`, `worker`, `db`, `result`) should reach the `Running` state.

## Accessing the App

**Minikube:**

```bash
minikube service vote
minikube service result
```

**Other clusters (NodePort):**

| App | URL |
|-----|-----|
| Vote | `http://<node-ip>:31000` |
| Result | `http://<node-ip>:31001` |

**Any cluster (port-forward):**

```bash
kubectl port-forward pod/vote 8080:80
kubectl port-forward pod/result 8081:80
```

Then open `http://localhost:8080` (vote) and `http://localhost:8081` (result).

## Troubleshooting

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs worker
```

## Cleanup

```bash
kubectl delete -f .
```

## Notes

- This is a learning project. It uses bare Pods, so there is no self-healing, replica management, or rolling updates. Deployments would be the next step.
- The Postgres credentials (`postgres` / `postgres`) are set as plain-text environment variables in `db-pod.yaml`. This is fine for a demo, but use Kubernetes Secrets for anything real.
- The Services are named `redis` and `db` so the app components can reach them by DNS name inside the cluster.

## Tech Stack

Kubernetes · Docker · Redis · PostgreSQL

## Author

[nikhil-flux](https://github.com/nikhil-flux)