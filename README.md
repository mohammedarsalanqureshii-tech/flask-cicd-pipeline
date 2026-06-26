# Flask CI/CD Pipeline on Kubernetes (K3s, AWS EC2)

A Flask application with a complete, automated CI/CD pipeline: every push to `main` builds a Docker image, pushes it to Docker Hub, and deploys it to a Kubernetes (K3s) cluster running on AWS EC2 — with health checks, resource limits, and rollback-safe image tagging.

## Architecture

```
Push to main
     │
     ▼
GitHub Actions
     │
     ├── Build Docker image (tagged with commit SHA + latest)
     ├── Push image to Docker Hub
     └── SSH into EC2 → kubectl set image
                              │
                              ▼
                    K3s cluster (AWS EC2)
                    ┌──────────────────────┐
                    │  Deployment (3 pods)  │
                    │  - resource limits    │
                    │  - liveness/readiness │
                    │    probes on /health  │
                    │  ──────────────────── │
                    │  Service (NodePort)   │
                    └──────────────────────┘
```

## What this project demonstrates

- Containerizing a Python Flask app with Docker
- Designing a CI/CD workflow in GitHub Actions from scratch
- Managing pipeline credentials (Docker Hub token, SSH key, EC2 host) entirely through GitHub Secrets — zero hardcoded credentials anywhere in the codebase
- Deploying and scaling an app on Kubernetes (K3s) with a Deployment and a NodePort Service
- Tagging Docker images with the Git commit SHA so every deployment is traceable and reversible, instead of relying on a single mutable `latest` tag
- Resource requests/limits and liveness/readiness probes, backed by a real `/health` endpoint
- Automating deployment rollouts on every push, with zero manual `kubectl` commands required in normal operation

## Tech stack

- **App:** Python, Flask
- **Containerization:** Docker
- **CI/CD:** GitHub Actions
- **Orchestration:** Kubernetes (K3s)
- **Infrastructure:** AWS EC2

## Project structure

```
.
├── app.py                       # Flask application (home + health endpoints)
├── requirements.txt              # Python dependencies (pinned)
├── Dockerfile                    # Container build instructions
├── deployment.yaml               # Kubernetes Deployment (3 replicas, probes, resource limits)
├── service.yaml                  # Kubernetes Service (NodePort)
├── .dockerignore                 # Files excluded from the Docker build context
├── .gitignore                    # Files excluded from version control
└── .github/workflows/
    └── docker-build.yml          # CI/CD pipeline definition
```

## How the pipeline works

1. Code is pushed to `main`.
2. GitHub Actions checks out the code and authenticates to Docker Hub using stored secrets.
3. The Docker image is built and pushed to Docker Hub, tagged with both `latest` and the Git commit SHA.
4. The workflow connects to the EC2 instance over SSH and runs `kubectl set image`, pointing the Deployment at the new SHA-tagged image — not just restarting pods with whatever `latest` happens to be.
5. Kubernetes performs a rolling update: new pods must pass the `/health` readiness probe before old pods are terminated, so the app stays available throughout the rollout.

## API endpoints

| Route     | Description                              |
|-----------|--------------------------------------------|
| `/`       | Returns app status and the pod's hostname |
| `/health` | Health check used by Kubernetes probes    |

## Running it locally

```bash
git clone https://github.com/mohammedarsalanqureshii-tech/flask-cicd-pipeline.git
cd flask-cicd-pipeline
docker build -t flask-cicd-pipeline .
docker run -p 5000:5000 flask-cicd-pipeline
```

Visit `http://localhost:5000` for the app and `http://localhost:5000/health` for the health check.

## Deploying to Kubernetes

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods
kubectl get svc
```

## Real issues debugged on this project

**CrashLoopBackOff from a missing health endpoint.** After adding liveness/readiness probes pointing at `/health`, new pods kept restarting. `kubectl logs` showed `GET /health` returning `404`, and `kubectl describe pod` confirmed it: `Liveness probe failed: HTTP probe failed with statuscode: 404`. The root cause wasn't the YAML — it was that the running container was still built from an older commit that didn't have the `/health` route yet, because the file hadn't actually been included in the prior `git commit` (caught via `git show <commit> --stat`, which showed the file was missing from that commit despite the commit message implying otherwise). Fixed by committing the correct file and verifying with `git status` before pushing.

**A `matchLabel` vs `matchLabels` typo.** A single missing letter in the Deployment's selector meant Kubernetes silently failed to associate the Service with the Pods — no error, just unexpected behavior. Fixed by carefully re-reading the YAML against the Kubernetes API reference.

## What I learned

This project was built to understand CI/CD and Kubernetes hands-on rather than through tutorials alone: debugging real selector mismatches, securing pipeline credentials properly, diagnosing a CrashLoopBackOff by tracing it from Kubernetes events back to a specific Git commit, and moving from a single `latest` tag to commit-SHA-based image tagging for safer, reversible deployments.

## Next steps

- Add automated tests that run in CI before the image is built
- Add monitoring with Prometheus and Grafana
- Provision the underlying AWS infrastructure with Terraform instead of manual setup
