# CI/CD Pipeline for a Flask App

CI/CD pipeline for a Flask app: GitHub Actions runs tests, builds a Docker image, and deploys to Kubernetes.

> **Status: in progress.** Test stage is live in CI. Build/push and deploy stages are being added next — see [Roadmap](#roadmap).

## What this project demonstrates

A small Flask API taken through a full CI/CD loop:

```
push to master
     │
     ▼
 run tests (pytest)  ─────┐
     │                    │  [not yet added]
     ▼                    ▼
 build Docker image   deploy to Kubernetes
     │
     ▼
 push to registry
```

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
    └── deploy.yml          # CI: install deps + run pytest on every push/PR to master
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

`.github/workflows/deploy.yml` currently runs on every push/PR to `master`:
1. Checks out the repo
2. Sets up Python 3.12
3. Installs dependencies from `app/requirements.txt`
4. Runs the pytest suite

Build, push-to-registry, and deploy-to-cluster stages are being added next (see Roadmap).

## Roadmap

- [ ] Add Docker build + push stage to CI (push image to Docker Hub on `master`)
- [ ] Add deploy stage to CI (apply `k8s/` manifests to a live cluster)
- [ ] Add a Jenkinsfile as an alternate CI/CD implementation
- [ ] Add architecture diagram
- [ ] Live-test `k8s/` manifests against a running cluster (minikube or GKE)

## Notes / issues hit along the way

Keeping this section honest rather than pretending it all worked first try — it's useful signal for anyone (including future me) debugging the same thing:

- **`pip` launcher pointed at a stale Python 3.12 path** while running Python 3.13 → fixed by using `python -m pip install ...` instead of `pip install ...` directly.
- **`ModuleNotFoundError: No module named 'flask'`** when running tests from the repo root → `requirements.txt` and `test_main.py` live in `app/`, needed to `cd app` first.
- **Docker Desktop engine not running** (`docker build` failed to connect to the daemon) → started Docker Desktop and waited for it to fully initialize before building.
- **`kubectl apply` failed with an OpenAPI validation connection error** → no local cluster (minikube) was running at the time; manifests were committed and will be validated against a live cluster later.
- **`git push` failed with `Permission denied (publickey)`** → remote was set to the SSH URL with no SSH key configured; switched remote to HTTPS instead.
- **Pushed to `main` but branch is `master`** → CI workflow trigger and push target both corrected to `master` to match the actual local branch.
