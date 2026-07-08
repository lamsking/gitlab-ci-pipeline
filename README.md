# GitLab CI/CD Pipeline — Static Website (Docker + Multi-Environment Deploy)

A four-stage GitLab CI pipeline — **Build → Test → Release → Deploy** — that packages an Nginx-based static website into a Docker image, smoke-tests it using Docker-in-Docker, pushes it to Docker Hub, and deploys it via SSH across three GitLab environments: **Review, Staging, and Production**.

## Stack
- GitLab CI/CD (Docker-in-Docker executor)
- Docker / Docker Hub
- Nginx
- SSH (remote deployment)

## Pipeline stages

```
Build (docker:dind) 
  → Test (docker:dind, curl smoke test) 
  → Release (push to Docker Hub) 
  → Deploy Review / Staging / Production (SSH, GitLab Environments)
```

### 1. Build
Runs inside `docker:dind` (Docker-in-Docker). Builds the image from the Dockerfile and saves it as a `.tar` artifact (`docker save`), so later stages don't need to rebuild — they just load the same image, guaranteeing what's tested is exactly what gets deployed.

### 2. Test
Loads the artifact image, runs it locally inside the CI job, and smoke-tests it with `curl -I` checking for a `200` response before tearing down.

### 3. Release
Logs into Docker Hub, loads the same tested image artifact, tags it under the Docker Hub namespace, and pushes it — reusing a YAML anchor (`.release_template`) to keep the job DRY.

### 4. Deploy (Review / Staging / Production)
Each environment reuses a shared `.deploy_template` anchor and:
- Sets up an SSH agent using a base64-encoded private key from a CI/CD variable
- Connects to the target server and: pulls the new image, removes the previous container, and runs the new one
- Uses GitLab's `environment:` keyword so each stage shows up as a tracked deployment (Review / Staging / Production) in GitLab's environment view

## Dockerfile

The image is a minimal Nginx-based static site container:
- Based on `nginx:${version}` (build-arg configurable, defaults to `latest`)
- Clears the default Nginx web root and clones the static site source directly into it at build time
- Exposes port 80 and runs Nginx in the foreground (`daemon off`)

## CI/CD variables required
| Variable | Type | Purpose |
|---|---|---|
| `DOCKER_PASSWORD` | Masked/protected | Docker Hub authentication |
| `SSH_PRIVATE_KEY` | Masked/protected, base64-encoded | SSH access to deployment servers |
| `SERVER_IP` (per environment) | Environment-scoped variable | Target server for Review/Staging/Production |

## How to run
Push to a GitLab repo with this `.gitlab-ci.yml` and `Dockerfile`, configure the CI/CD variables above (scoped per environment for `SERVER_IP`), and the pipeline runs automatically on push — building, testing, releasing, and deploying through each environment in order.

## What I'd improve next
- Add manual approval gates (`when: manual`) on Staging/Production deploy jobs, so promotion isn't fully automatic
- Tag Docker images with the CI commit SHA instead of a static `v1`, to keep a real deployment history
- Add a rollback job that redeploys the previous image tag if the health check fails
- Pull the git-cloned static site in as a build context / CI artifact instead of cloning it live inside the Dockerfile, so builds aren't dependent on an external repo's availability
