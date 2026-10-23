# k8s-inventory-kind

An inventory API and Postgres database running on a local Kubernetes cluster (kind), configured the way a production service would be: configuration separate from the image, health probes, self-healing, zero-downtime updates and a non-root container.

## Architecture

~~~mermaid
flowchart LR
  client["curl / load pod"] --> svc["Service: api<br/>ClusterIP :80"]
  svc --> api["Deployment: api<br/>2 replicas, rolling updates"]
  api --> dbsvc["Service: db"]
  dbsvc --> pg[("StatefulSet: db-0<br/>Postgres 16 + 1Gi volume")]
  cm["ConfigMap: api-config"] -.-> api
  sec["Secret: db-credentials<br/>created by command, not in git"] -.-> api
  sec -.-> pg
~~~

## What it demonstrates

- **Persistent state:** Postgres runs as a StatefulSet with its own volume, and data survives pod deletion.
- **Configuration and secrets:** settings live in a ConfigMap, and the password in a Secret that is created by a command and never committed.
- **Two kinds of health check:** readiness uses `/health`, which also checks the database, so a pod that cannot reach Postgres stops receiving traffic. Liveness uses a plain TCP check, so a database outage does not cause every API pod to restart.
- **Safe updates:** a rolling update with `maxUnavailable: 0` replaces pods one at a time, and `kubectl rollout undo` reverts a bad change.
- **Hardening:** the container runs as a non-root user, cannot escalate privileges, drops all Linux capabilities, and has resource requests and limits.
- **Self-healing:** deleting a pod makes the Deployment create a replacement.

## Run it

The API source and its Dockerfile live in [inventory-app](https://github.com/fiehna-hue/inventory-app).

~~~
git clone https://github.com/fiehna-hue/inventory-app.git
docker build -t inventory-api:local inventory-app
kind create cluster --name inventory
kind load docker-image inventory-api:local --name inventory
kubectl apply -f k8s/00-namespace.yaml
kubectl create secret generic db-credentials --namespace inventory --from-literal=POSTGRES_PASSWORD="$(openssl rand -base64 18 | tr -d '/+=')"
kubectl apply -f k8s/10-postgres.yaml
kubectl wait --for=condition=ready pod/db-0 -n inventory --timeout=120s
kubectl apply -f k8s/20-api-config.yaml -f k8s/30-api.yaml
~~~

Try it:

~~~
kubectl port-forward -n inventory svc/api 8080:80 &
curl -s localhost:8080/health
curl -s -X POST localhost:8080/items -H 'Content-Type: application/json' -d '{"name":"kind-test","quantity":2}'
curl -s localhost:8080/items
~~~

Clean up with `kind delete cluster --name inventory`.

## Evidence

The API answered through the Service, wrote to Postgres and read the row back:

~~~
$ curl -s localhost:8080/health
{"status":"ok"}
$ curl -s localhost:8080/items
[{"id":1,"name":"kind-test","quantity":2}]
~~~

The container runs as a non-root user:

~~~
$ kubectl exec -n inventory deploy/api -- id
uid=10001(appuser) gid=10001(appuser) groups=10001(appuser)
~~~

A table created in Postgres was still there after the pod was deleted and recreated. A rolling update was started and then reverted with `kubectl rollout undo`, which removed the extra environment variable it had added.

## Incidents

Three failures I caused on purpose and diagnosed, with symptom, cause, fix and lesson: [docs/incident-log.md](docs/incident-log.md).

## Known limitations

- Local kind cluster only, so there is no cloud load balancer, DNS or TLS.
- The image is loaded into the cluster by hand as `inventory-api:local`. A real setup would pull from a registry such as ECR.
- A single Postgres replica with no backups.
- The Secret is created manually. A production setup would use an external secrets manager.
- No NetworkPolicy, Ingress or autoscaling yet.

## Next

An Ingress, a HorizontalPodAutoscaler, kustomize overlays, and the same workload on EKS.
