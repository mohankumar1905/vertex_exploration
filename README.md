# Vertex Exploration: FastAPI CI/CD to Vertex AI

## Overview
This sample project demonstrates how to build a simple **FastAPI** service, containerize it with Docker, and set up a **GitHub Actions** CI/CD pipeline that automatically builds the Docker image, pushes it to **Google Artifact Registry**, and deploys it to **Vertex AI** as a custom container model.

## Project Structure
```
vertex_exploration/
├─ app/
│  └─ main.py          # FastAPI application
├─ .github/
│  └─ workflows/
│     └─ ci_cd.yml     # GitHub Actions workflow
├─ Dockerfile           # Build image for Vertex AI
├─ requirements.txt     # Python dependencies
├─ .gitignore
└─ README.md            # This file
```

## Prerequisites
1. **Google Cloud Project** with Vertex AI API enabled.
2. **Artifact Registry** repository created (e.g., `vertex-repo`).
3. **Workload Identity Federation** set up to allow GitHub Actions to obtain short‑lived access tokens. You will need the following GitHub secrets in your repository:
   - `GCP_WIP` – Workload Identity Provider resource name.
   - `GCP_SA_EMAIL` – Service Account email with `roles/aiplatform.admin` and `roles/artifactregistry.writer`.
   - `GCP_PROJECT_ID` – Your GCP project ID.
   - `GCP_REGION` – Region for Vertex AI (e.g., `us-central1`).
4. A **GitHub repository** where this code will be pushed.

## Local Development
```bash
# Clone the repo (or copy the folder locally)
git clone <your-repo-url>
cd vertex_exploration

# Create a virtual environment (optional but recommended)
python -m venv .venv
source .venv/bin/activate  # on Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Run the FastAPI app locally
uvicorn app.main:app --host 0.0.0.0 --port 8080
```
Open your browser at `http://localhost:8080/` – you should see `{"message": "Hello, Vertex AI!"}`.

## CI/CD Pipeline (GitHub Actions)
The workflow (`.github/workflows/ci_cd.yml`) runs on every push to the `main` branch and performs the following steps:
1. **Checkout** the repository.
2. **Set up Python** (3.11) and install dependencies.
3. **Authenticate to Google Cloud** using the `google-github-actions/auth` action with Workload Identity.
4. **Configure Docker** to push to Artifact Registry.
5. **Build** the Docker image.
6. **Push** the image to Artifact Registry.
7. **Upload** the image as a Vertex AI custom container model.
8. **Deploy** the model to a Vertex AI endpoint (creates the endpoint if it does not exist).

### How Secrets are Used
- `GCP_WIP` and `GCP_SA_EMAIL` let the `auth` action obtain an **access token**.
- `GCP_PROJECT_ID` and `GCP_REGION` are used throughout the script to reference the correct project and region.

## Deploying Manually (Optional)
If you prefer to deploy outside of GitHub Actions, you can run the same commands locally after authenticating with `gcloud`:
```bash
# Authenticate with gcloud (you need the service account key locally)
gcloud auth activate-service-account --key-file=path/to/key.json

gcloud config set project $GCP_PROJECT_ID

gcloud auth configure-docker $GCP_REGION-docker.pkg.dev

docker build -t $IMAGE_TAG .

docker push $IMAGE_TAG

# Upload model
MODEL_NAME="fastapi-manual-$(date +%s)"
gcloud ai models upload \
  --region=$GCP_REGION \
  --display-name=$MODEL_NAME \
  --container-image-uri=$IMAGE_TAG \
  --container-health-route="/" \
  --container-predict-route="/"

# Deploy to endpoint (create if needed)
ENDPOINT_NAME="fastapi-endpoint"
ENDPOINT_RESOURCE=$(gcloud ai endpoints list --region=$GCP_REGION --filter="displayName=$ENDPOINT_NAME" --format="value(name)")
if [ -z "$ENDPOINT_RESOURCE" ]; then
  ENDPOINT_RESOURCE=$(gcloud ai endpoints create --region=$GCP_REGION --display-name=$ENDPOINT_NAME --format="value(name)")
fi

MODEL_RESOURCE=$(gcloud ai models list --region=$GCP_REGION --filter="displayName=$MODEL_NAME" --format="value(name)")

gcloud ai endpoints deploy-model $ENDPOINT_RESOURCE \
  --region=$GCP_REGION \
  --model=$MODEL_RESOURCE \
  --display-name="fastapi-deployment-$(date +%s)" \
  --machine-type=n1-standard-2 \
  --traffic-split=0=100
```

## Next Steps
- **Push** this repository to GitHub and add the required secrets.
- Verify the **GitHub Actions** run completes successfully.
- Once deployed, you can send HTTP requests to the Vertex AI endpoint URL (found in the endpoint details) to invoke the FastAPI service.

---
*Feel free to ask if you need help setting up the GCP resources, configuring secrets, or troubleshooting the pipeline.*
