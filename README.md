# CI/CD Pipeline for a Flask App

CI/CD pipeline for a Flask app: GitHub Actions runs tests, builds a Docker image, and deploys to Kubernetes.

## What this project demonstrates

A small Flask API taken through a full CI/CD loop, end to end:

```
push to master
     │
     ▼
 test (pytest)
     │  fails → pipeline stops here
     ▼
 build-and-push (Docker image, tagged with commit SHA, pushed to Docker Hub)
     │
     ▼
 deploy (spins up an ephemeral kind cluster, deploys the just-built image,
         waits for rollout to succeed, prints running pods)
```

Every stage depends on the previous one succeeding (`needs:` in the workflow), so a broken test or a bad image never reaches deployment.

## Stack

- **App:** Python 3.12 / Flask
- **Tests:** pytest
- **Containerization:** Docker
- **Orchestration:** Kubernetes (Deployment + Service manifests)
- **CI/CD:** GitHub Actions

## Project structure

```
ci-cd-flask-k8s-pipeline/
├── app/
│   ├── main.py           # Flask app with / and /health endpoints
│   ├── test_main.py      # unit tests for both endpoints
│   └── requirements.txt
├── Dockerfile
├── .dockerignore
├── k8s/
│   ├── deployment.yaml    # 2 replicas, resource requests/limits, readiness/liveness probes
│   └── service.yaml       # ClusterIP service on port 80 -> 5000
└── .github/workflows/
    └── deploy.yml          # 3-stage pipeline: test -> build-and-push -> deploy
```

## Running locally

```bash
cd app
python -m pip install -r requirements.txt
pytest test_main.py -v
python main.py
```

Then check `http://localhost:5000/health` returns `{"status": "ok"}`.

## Running in Docker

```bash
docker build -t ci-cd-flask-k8s-pipeline .
docker run -p 5000:5000 ci-cd-flask-k8s-pipeline
curl http://localhost:5000/health
```

## Kubernetes manifests

`k8s/deployment.yaml` and `k8s/service.yaml` define a 2-replica deployment with:
- **Resource requests/limits** — so the scheduler can bin-pack properly and one pod can't starve its neighbours.
- **Readiness probe** — controls whether the pod receives traffic.
- **Liveness probe** — controls whether Kubernetes restarts the pod if it's stuck.

Apply against any cluster (once an image exists in a registry):
```bash
kubectl apply -f k8s/
kubectl get pods
kubectl get svc
```

## CI pipeline

`.github/workflows/deploy.yml` runs on every push/PR to `master`, as three dependent jobs:

**1. `test`** — runs on every push and PR
- Checks out the repo
- Sets up Python 3.12
- Installs dependencies from `app/requirements.txt`
- Runs the pytest suite

**2. `build-and-push`** — runs only on pushes to `master` (not PRs), and only if `test` passes
- Logs in to Docker Hub using repo secrets
- Builds the image and pushes it tagged with the commit SHA (`ci-cd-flask-k8s-pipeline:<sha>`), not `latest` — every build is traceable back to the exact commit that produced it

**3. `deploy`** — runs only if `build-and-push` succeeds
- Spins up a throwaway `kind` (Kubernetes-in-Docker) cluster inside the runner
- Swaps the image placeholder in `k8s/deployment.yaml` for the image just built
- Applies the manifests and waits on `kubectl rollout status`, so the job actually fails if the pods don't come up healthy — not just if `kubectl apply` didn't error
- Logs `kubectl get pods` as visible proof of a working deployment

Because each job uses `needs:`, a failing test never reaches build, and a failed build never reaches deploy.

## Triggering an ephemeral AKS environment

After a successful deploy, the pipeline fires a `repository_dispatch` event
at [`aks-ephemeral-infra`](https://github.com/Reeceakhun/aks-ephemeral-infra) —
a separate repo that owns the full lifecycle of short-lived Azure
environments. That repo provisions a tagged AKS cluster, deploys the image
just built here, and automatically tears it down ~20 minutes later via a
scheduled reaper — independent of whether this pipeline (or anything else)
is still running.

This repo doesn't manage any Azure infrastructure directly; it only asks for
an environment and moves on. See `aks-ephemeral-infra`'s README for the full
architecture, tagging convention, and OIDC setup.

## Design decisions

- **Ephemeral `kind` cluster instead of a persistent cloud cluster** — this keeps the pipeline fully self-contained and free to run: anyone who forks this repo can run the whole thing without needing GCP/AWS credentials or a standing cluster. (See `reece-project` for a persistent GKE cluster provisioned with Terraform.)
- **Commit-SHA image tags over `latest`** — makes every deployed image traceable to an exact commit, and avoids the classic "which version of `latest` is actually running" problem.
- **`readiness`/`liveness` probes on `/health`** — readiness gates traffic to the pod, liveness controls restarts. Using the same endpoint for both here for simplicity; in a larger app these would typically check different things (e.g. liveness just checks the process is alive, readiness checks DB connectivity).

## Roadmap

- [ ] Add a Jenkinsfile as an alternate CI/CD implementation
- [ ] Add architecture diagram
- [ ] Point `deploy` at a persistent cluster (e.g. the GKE cluster from `reece-project`) as an alternative to the ephemeral `kind` job

## Notes / issues hit along the way

Keeping this section honest rather than pretending it all worked first try — it's useful signal for anyone (including future me) debugging the same thing:

- **`pip` launcher pointed at a stale Python 3.12 path** while running Python 3.13 → fixed by using `python -m pip install ...` instead of `pip install ...` directly.
- **`ModuleNotFoundError: No module named 'flask'`** when running tests from the repo root → `requirements.txt` and `test_main.py` live in `app/`, needed to `cd app` first.
- **Docker Desktop engine not running** (`docker build` failed to connect to the daemon) → started Docker Desktop and waited for it to fully initialize before building.
- **`kubectl apply` failed with an OpenAPI validation connection error** → no local cluster (minikube) was running at the time; manifests were committed and will be validated against a live cluster later.
- **`git push` failed with `Permission denied (publickey)`** → remote was set to the SSH URL with no SSH key configured; switched remote to HTTPS instead.
- **Pushed to `main` but branch is `master`** → CI workflow trigger and push target both corrected to `master` to match the actual local branch.
- **First CI run failed immediately with "job was not started because your account is locked due to a billing issue"** → account-level GitHub billing lock, unrelated to the workflow itself; resolved via GitHub billing settings.
- **Cross-repo trigger silently "succeeded" despite a failed API call** — the original `curl` command had no `-f` flag, so a `403` from GitHub's API still showed as a green step. Fixed by adding `-f` (fail on HTTP errors) and printing the response status, which surfaced the real error immediately.
- **`403: Resource not accessible by personal access token`** when firing the `repository_dispatch` event → the fine-grained PAT was missing `Contents: Read and write` — this endpoint requires that permission, not just `Actions`, which isn't obvious from the API's name. Fixed by updating the token's scopes.