# Waymate on Kubernetes (local practice)

Kubernetes manifests that run [Waymate](https://github.com/Dynamo285K/Waymate), a carpooling app, in a local [kind](https://kind.sigs.k8s.io/) cluster: the Bun/Elysia API, the React web app, PostgreSQL and Redis.

This is a learning project. Production Waymate runs on Render (API) and Cloudflare Pages (web). The goal here was to run the same images on Kubernetes and practise deployments, probes, configuration, rollouts and debugging.

## Architecture

```
Browser ── http://localhost:8080 (kubectl port-forward)
             │
        Service web ──► web pods ×2 (nginx, non-root)
                           ├── /api/auth/*  → Service api  (prefix kept, Better Auth)
                           ├── /api/*       → Service api  (prefix stripped)
                           └── /*           → index.html   (SPA fallback)
                                   │
                        Service api ──► api pods ×3 (Bun, non-root)
                                         ├── Service postgres ──► postgres pod
                                         └── Service redis    ──► redis pod

Jobs: migrate (Drizzle migrations), seed (dev fixtures)
```

The cluster has one control-plane node and two workers, so replicas spread across nodes.

## Layout

| Path | Contents |
| --- | --- |
| `kind-config.yaml` | Cluster: 1 control plane + 2 workers |
| `manifests/00-namespace.yaml` | `waymate` namespace |
| `manifests/01-secret.yaml` | Database password and URL, auth secret, API keys. **Practice values only.** |
| `manifests/10-postgres.yaml` | PostgreSQL Deployment + Service, readiness probe `pg_isready` |
| `manifests/11-redis.yaml` | Redis Deployment + Service (rate limiter store) |
| `manifests/20-configmap.yaml` | Non-secret API configuration |
| `manifests/30-migrate-job.yaml` | Job: applies Drizzle migrations |
| `manifests/31-seed-job.yaml` | Job: wipes the database and loads dev fixtures |
| `manifests/40-api.yaml` | API Deployment (3 replicas) + Service |
| `manifests/50-web.yaml` | nginx config (ConfigMap) + web Deployment (2 replicas) + Service |
| `web/Dockerfile` | Web image: Vite build, served by non-root nginx |
| `k8s-tahak.md` | Personal kubectl cheat sheet |

## Images

| Image | Built from | Used by |
| --- | --- | --- |
| `waymate-api` | Waymate `Dockerfile` (final stage: Bun + bundled `index.js`) | `api` Deployment |
| `waymate-migrate` | Waymate `Dockerfile`, `--target build` (full source + dev dependencies) | `migrate` and `seed` Jobs |
| `waymate-web` | `web/Dockerfile` in this repo, built from the Waymate repo root | `web` Deployment |

The tags must match the `image:` fields in the manifests. kind nodes cannot see the local Docker images, so every image is copied into the cluster with `kind load`.

## Run it

Prerequisites: Docker, [kind](https://kind.sigs.k8s.io/), kubectl, and a clone of the Waymate repo.

```bash
WAYMATE=~/Developer/spolujazda-sketch   # path to the Waymate repo

# 1. Cluster
kind create cluster --name waymate --config kind-config.yaml

# 2. Images (tags as in the manifests)
docker build -t waymate-api:v2 "$WAYMATE"
docker build --target build -t waymate-migrate:v1 "$WAYMATE"
docker build -f web/Dockerfile -t waymate-web:v1 "$WAYMATE"
kind load docker-image waymate-api:v2 waymate-migrate:v1 waymate-web:v1 --name waymate

# 3. Namespace, secrets, databases
kubectl apply -f manifests/00-namespace.yaml -f manifests/01-secret.yaml \
              -f manifests/10-postgres.yaml -f manifests/11-redis.yaml
kubectl config set-context --current --namespace=waymate
kubectl wait --for=condition=available deployment/postgres deployment/redis --timeout=120s

# 4. Config, migrations, fixtures (Kubernetes does not order these, so wait between steps)
kubectl apply -f manifests/20-configmap.yaml -f manifests/30-migrate-job.yaml
kubectl wait --for=condition=complete job/migrate --timeout=120s
kubectl apply -f manifests/31-seed-job.yaml
kubectl wait --for=condition=complete job/seed --timeout=180s

# 5. API and web
kubectl apply -f manifests/40-api.yaml -f manifests/50-web.yaml
kubectl rollout status deployment/api
kubectl rollout status deployment/web

# 6. Open http://localhost:8080 and sign in as admin@example.com / admin1234
kubectl port-forward svc/web 8080:80
```

Check the API directly:

```bash
kubectl port-forward svc/api 3000:3000
curl -s localhost:3000/health   # {"status":"ok","db":"up","commit":"v1"}
```

Jobs are immutable. To run one again, delete it first: `kubectl delete job seed`.

Clean up: `kind delete cluster --name waymate`.

## Design decisions

- **Readiness vs liveness.** The API readiness probe calls `/health`, which returns 503 when the database is down, so the pod only leaves the Service. Liveness is a TCP check, so a database outage never restarts the API.
- **Migrations as a Job**, not on API start-up: three replicas would otherwise migrate at the same time.
- **Configuration outside the images.** The API reads env vars from a ConfigMap and a Secret, and nginx reads its config from a ConfigMap. Env vars are only read at start, so a config change needs `kubectl rollout restart`.
- **Same-origin `/api` proxy in nginx**, the same routing as local Vite dev. The browser sees one origin, so no CORS and simpler auth cookies.
- **Non-root containers** for the API (`USER bun`) and the web (`nginx-unprivileged`, port 8080).
- **Memory limits without CPU limits.** Memory over the limit means OOMKilled (exit 137), CPU over a limit means throttling.

## Exercises done

- Self-healing: delete a pod, the Deployment replaces it
- Scaling: `kubectl scale deployment/api --replicas=5`
- Rolling update to a new image tag and `kubectl rollout undo`
- Database down: API pods stay `Running` but `0/1` ready, no restarts
- OOMKilled with a low memory limit (exit code 137)
- `ImagePullBackOff` with a non-existent tag

## Limitations and possible next steps

Shortcuts taken for a local cluster, and what a production setup would use instead:

| Now | Production / next step |
| --- | --- |
| PostgreSQL as a Deployment without a volume: data is lost when the pod restarts | Managed database (RDS, as Supabase is today), or a StatefulSet with a PersistentVolumeClaim (EBS on AWS) |
| Redis without persistence or auth | Managed Redis (ElastiCache) or a StatefulSet with a password |
| Secret values committed in plain YAML | External Secrets Operator with AWS Secrets Manager, or Sealed Secrets; encryption at rest for etcd |
| `kind load` instead of a registry | Push images to a registry (ECR / GitLab Container Registry), tag by commit SHA |
| `kubectl port-forward` | Ingress or Gateway API with TLS (cloud-provider-kind locally, AWS Load Balancer Controller + ALB on EKS) |
| Manual `kubectl apply` | CI pipeline: build once, push, run the migrate Job, then roll out. Or GitOps with Argo CD |
| Plain manifests with hard-coded values | Kustomize overlays or a Helm chart per environment (local, staging, production) |
| Fixed replica count | HorizontalPodAutoscaler on CPU, plus a PodDisruptionBudget for the API |
| Default scheduling | `topologySpreadConstraints` so replicas spread across nodes and zones |
| No network restrictions | NetworkPolicies: only the API may reach PostgreSQL and Redis |
| No monitoring | Prometheus + Grafana, central logs (Loki or CloudWatch) |
| Migration image contains the full source and dev dependencies | Slim migration stage with only Drizzle Kit and the SQL files |
| Web build args baked per environment | Runtime config (e.g. `config.js` served by nginx) so one web image fits every environment |
