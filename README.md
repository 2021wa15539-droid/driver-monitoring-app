# driver-monitoring-app — dev & deploy notes

Lightweight Flask app packaged as a Docker image and deployed to EKS. This repository contains a simple app in `src/`, a Dockerfile in `docker/`, a Jenkins pipeline in `jenkins/`, and a Kubernetes manifest in `k8s/`.

Local development

- Create a virtualenv and run locally (PowerShell):

```powershell
python -m venv .venv; .\.venv\Scripts\Activate; pip install -r requirements.txt
python src\app.py
```

- The Flask app listens on port 80 inside the container. For local Docker runs map to a higher host port (example maps host 8080 to container 80):

```powershell
docker build -f docker/Dockerfile -t eks-automation-repo:local .
docker run --rm -p 8080:80 eks-automation-repo:local
```

CI / CD (Jenkins -> ECR -> EKS)

- The pipeline is `jenkins/Jenkinsfile`. Key expectations:
  - Jenkins should run on an agent with Docker, AWS CLI and kubectl installed.
  - Provide or set `AWS_ACCOUNT_ID` in the environment (the Jenkinsfile contains a placeholder).
  - `IMAGE_TAG` is derived from the Jenkins build number. The pipeline:
    1. Builds the image from the repo root using `docker/Dockerfile`.
    2. Tags and pushes to ECR: `${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}`
    3. Updates the deployment in-cluster with `kubectl set image ...` to point the `driver-monitoring` container at the newly pushed tag.

Notes and gotchas

- Dockerfile location vs build context: the file is `docker/Dockerfile`. The pipeline now builds from the repo root and passes `-f docker/Dockerfile` to ensure the Dockerfile is found.
- k8s manifest image placeholder: `k8s/deployment.yaml` contains templated placeholders. The Jenkins pipeline updates the image using `kubectl set image`, so you don't need to modify the file for automated deploys.
- Fill in `AWS_ACCOUNT_ID` (and ensure IAM/credentials for pushing to ECR and updating the EKS cluster) before running the pipeline.

If you want me to wire in parameterized Jenkins credentials or add a small `make` or PowerShell script for local dev automation, tell me which you'd prefer and I'll add it.
