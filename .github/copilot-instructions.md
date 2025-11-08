## Quick context (what this repo is)

- Small Flask service (single file) serving a hello page: `src/app.py`.
- Containerized via `docker/Dockerfile`. Dependencies listed in `requirements.txt`.
- CI/CD: `jenkins/Jenkinsfile` builds a Docker image, pushes to AWS ECR and deploys to EKS via `k8s/deployment.yaml`.

## Big-picture architecture & why it matters

- One lightweight web service packaged as a Docker image and deployed to EKS.
- The pipeline assumes an AWS-centric workflow: Jenkins -> ECR -> EKS. When editing automation, update Jenkinsfile and the k8s manifest together.
- Image name / account-id and AWS region are templated/placeholder values in `jenkins/Jenkinsfile` and `k8s/deployment.yaml` — these must be filled with real values during setup.

## Important, repo-specific details (must-know)

- Dockerfile location vs build context: the Dockerfile is at `docker/Dockerfile` but the Jenkinsfile runs `docker build -t $ECR_REPO:latest ./src` (build context `./src`). This means the pipeline as-written will fail unless you either move `Dockerfile` into `src/` or change the build command to `docker build -f docker/Dockerfile -t $ECR_REPO:latest .`.
- The Dockerfile copies `requirements.txt` and `src/app.py` — those paths only exist in the build context if you use the repo root as the build context (see above mismatch).
- The Flask app binds to 0.0.0.0:80. When running locally without sudo privileges, map ports (host 8080 -> container 80) or change the app port for development.
- k8s `deployment.yaml` references the ECR image with a placeholder `<your-account-id>`; ensure this matches the ECR repository used by the pipeline.

## Developer workflows (concrete commands & examples)

- Run locally (Python, no Docker):

  - Create virtualenv & install:

    ```powershell
    python -m venv .venv; .\.venv\Scripts\Activate; pip install -r requirements.txt
    python src\app.py
    ```

  - If port 80 requires privileges, run on a higher port by editing `app.run(host='0.0.0.0', port=80)` or by running in Docker and mapping ports.

- Build & run via Docker (recommended for parity with CI):

  - From repo root (preferred):

    ```powershell
    docker build -f docker/Dockerfile -t eks-automation-repo:latest .
    docker run --rm -p 8080:80 eks-automation-repo:latest
    ```

  - Jenkins pipeline currently uses `./src` as build context; if you keep that, ensure `docker/Dockerfile` is moved to `src/` or adapt the pipeline.

- CI/CD notes: `jenkins/Jenkinsfile` expects:
  - AWS credentials/config in the Jenkins agent environment
  - `AWS_REGION`, `ECR_REPO` and AWS account id set correctly
  - `kubectl` configured to point at the target EKS cluster

## Patterns & conventions observed

- Minimal, single-module Flask app. Expect any new features to follow the same pattern: small modules added under `src/` and imported by `app.py`.
- Image tagging: the pipeline uses the `latest` tag only. When adding releases, consider introducing semantic tags and updating `k8s/deployment.yaml` to use specific tags.

## Integration points & external dependencies

- AWS ECR (image registry) — credentials and account id required.
- EKS (kubectl apply) — Jenkins agent must have kubeconfig or IAM role to update the cluster.
- Jenkins — pipeline is groovy in `jenkins/Jenkinsfile`. It contains hard-coded repo URL with a username-like string; confirm this is the intended remote.

## Troubleshooting tips for agents

- If a build step fails in Jenkins, first check the Docker build context vs Dockerfile path (common cause here).
- If the container starts but `kubectl apply -f k8s/deployment.yaml` fails, check image name placeholders and the Jenkins agent's kubeconfig/permissions.

## Files to reference when making changes

- `src/app.py` — service entrypoint and port binding.
- `docker/Dockerfile` — image contents, Python version (3.9-slim).
- `requirements.txt` — pinned dependencies (Flask==2.1.0).
- `jenkins/Jenkinsfile` — CI/CD pipeline (build, push, deploy).
- `k8s/deployment.yaml` — deployment replica count, labels, and container image.

## If you need to extend or refactor

- Keep build context and Dockerfile paths consistent. Prefer using the repo root as the build context and `-f` to point to the Dockerfile.
- Prefer explicit image tags for deployments to avoid surprises with `latest`.

---

If any of the placeholders (AWS account id, ECR repo name, or Jenkins remote URL) are intentional test values, tell me which values should remain templated and I will keep them as variables in future edits.
