# Lightweight model for sorting waste

This repository is a group project developed for the course Machine Learning Operations (02476) at The Technical University of Denmark (DTU).

The project aims at finetuning an open source model for classification of images of waste into categories of recyclable materials. Focus will be kept on a flexible solution, that utilizes a range of MLOps technologies

## Project Description
Our customer requests a lightweight image classification solution capable of categorizing images of waste into a fixed set of recycling categories. The solution must be able to run locally on mobile devices with limited hardware resources. Additionally, it should be easy to extend the system with new recycling categories in order to adapt to changes in recycling regulations and differences across markets.

To accommodate these requirements, we will leverage the skills acquired during the Machine Learning Operations course to develop a proof-of-concept deep learning solution that is both robust and easily adaptable. The system will take an image of a single piece of waste as input and output an estimate of the appropriate recycling category.

Initially, the recycling categories will include paper, glass, plastic, metal, cardboard, and non-recyclable waste. However, the system should be designed to easily support additional categories, such as hazardous waste, batteries, or colored versus clear glass, without requiring major architectural changes.

As a baseline, we will fine-tune a pretrained PyTorch-based image classification model, MobileNet-v3, from the PyTorch Image Models (timm) library, with a focus on lightweight architectures suitable for mobile deployment with the possibility of investigating other pretrained models if time allows (e.g., MobileNet variants). The model will be trained on the wasteNet dataset hosted on Hugging Face, which contains approximately 5,000 labeled images across the initial recycling categories. The solution will be designed to accommodate new training data and enable experimentation with different base models in a flexible and reproducible manner.

In our implementation, we will focus on utilizing tools for stable cloud-deployment such as Docker and FastAPI. Furthermore we will make use of tools for continuous integration (CI) and experiment tracking in order to ensure reproducibility, maintanability and scalability.

### Resources
- [wastenet dataset](https://huggingface.co/datasets/garythung/wastenet)
- [MobileNet-v3](https://huggingface.co/timm/mobilenetv3_small_100.lamb_in1k)


## Installation
The easiest way to get started is using the provided dev container which includes all dependencies pre-configured.

1. **Prerequisites:**
   - [Docker Desktop](https://www.docker.com/products/docker-desktop/)
   - [VS Code](https://code.visualstudio.com/) with [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

2. **Setup:**
   ```bash
   # Clone the repository
   git clone https://github.com/madsoedk/waste-image-sorting.git
   cd waste-image-sorting

3. Open in Dev Container:
   Open the project in VS Code
   Press F1 and select "Dev Containers: Reopen in Container"
   Wait for the container to build and dependencies to install (this may take a few minutes the first time)

## Setup before training and deployment
1. Install Google Cloud SDK (skip if project is opened with `.devcontainer/devcontainer.json` that automatically installs the SDK): [https://docs.cloud.google.com/sdk/docs/install-sdk](https://docs.cloud.google.com/sdk/docs/install-sdk)
2. Docker should be installed, check by running the command `docker` in your terminal, otherwise install docker on your system.
3. Authenticate and setup
```
# Login to GCP
gcloud auth login

# Set your project ID
gcloud config set project wasteclassification-484408

# Enable required APIs
gcloud services enable artifactregistry.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable compute.googleapis.com
gcloud services enable aiplatform.googleapis.com
gcloud services enable storage.googleapis.com

# Configure Docker authentication
gcloud auth configure-docker europe-west1-docker.pkg.dev
```
4. Create `.env` file with environment variables (see `.env.example`)

### Workflows for building training and API images

Both training and API images are automatically built and pushed to the GCP registry when changes are made to the main branch. They can also be triggered manually or through a release by running the following (or activating in GitHub Actions):

**Manual trigger of training image:**
```bash
gh workflow run build-train-image.yaml
gh workflow run build-train-image.yaml -f tag=custom-tag
```

**Manual trigger of API image:**
```bash
# Build only
gh workflow run build-api-image.yaml

# Build and deploy
gh workflow run build-api-image.yaml -f deploy=true

# Build with custom tag
gh workflow run build-api-image.yaml -f tag=staging -f deploy=false
```

**Creating a release:**
```bash
# Create a release (triggers automatically)
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
gh release create v1.0.0 --title "v1.0.0" --notes "Release notes"

# Or trigger manually
gh workflow run build-on-release.yaml -f version=v1.0.0
```

## Testing

The project includes comprehensive test coverage for unit tests, API integration tests, and load/performance tests.

### Running Tests

```bash
# Run all tests
pytest tests/ -v

# Run unit tests only
pytest tests/unittests/ -v

# Run API integration tests
pytest tests/integrationtests/ -v

# Run with coverage
coverage run -m pytest tests/
coverage report -m
```

### API Load Testing

Load tests measure API performance under different loads using Locust:

```bash
# 1. Start the API server
uvicorn wastesorting.api:app --host 0.0.0.0 --port 8000

# 2. Run load tests (in a new terminal)
./run_load_tests.sh

# Custom configuration
./run_load_tests.sh --users 100 --spawn-rate 10 --time 5m

# Run with web UI for real-time monitoring
./run_load_tests.sh --web
```

### Continuous Integration

All tests run automatically in GitHub Actions on every push and pull request:
- **Unit Tests** - Test individual functions and classes
- **Integration Tests** - Test API endpoints and their integration
- **Load Tests** - Run on main branch to verify performance

## Deployment
Complete installation in [setup section](#Setup-bfore-training-ad-deployment) first to make sure you have the required dependencies.

### Train in Google Cloud Vertex
1. Build and push docker images
```bash
# Build the train-image for GCP
uv run invoke gcp-build-train

# Push to Artifact Registry
uv run invoke gcp-push-train
```

2. Submit training job

```bash
# Basic training job (CPU only)
invoke gcp-train-vertex

# Example with custom settings
invoke gcp-train-vertex \
    --job-name my-training-job \
    --bucket-name your-bucket-name \
    --machine-type n1-standard-4


```

To follow custom training job follow instructions in terminal or find JOB_ID through:

```bash
# List jobs
gcloud ai custom-jobs list --region=europe-west1
```
And insert in the following:

```bash
# View job details
gcloud ai custom-jobs describe JOB_ID --region=europe-west1
```

### Deploy to Google Cloud Run
1. Build and push docker images
```
# Build the API-image for GCP
uv run invoke gcp-build-api

# Push to Artifact Registry
uv run invoke gcp-push-api
```
2. Deploy to Cloud Run
```
gcloud run deploy wastesorting-api \
  --image europe-west1-docker.pkg.dev/wasteclassification-484408/wasteclassification/api:latest \
  --platform managed \
  --region europe-west1 \
  --allow-unauthenticated \
  --memory 2Gi \
  --cpu 2 \
  --timeout 300 \
  --port 8000
```
3. Get Service URL
```
gcloud run services describe wastesorting-api \
  --region europe-west1 \
  --format 'value(status.url)'
```
4. Test deployed API (Give it a minute or two to boot)
```
# Set service URL
export SERVICE_URL=$(gcloud run services describe wastesorting-api \
  --region europe-west1 \
  --format 'value(status.url)')

curl ${SERVICE_URL}/api/health
```

## Project structure

The directory structure of the project looks like this:
```txt
├── .github/                  # Github actions and dependabot
│   ├── dependabot.yaml
│   └── workflows/
│       └── tests.yaml
├── configs/                  # Configuration files
├── data/                     # Data directory
│   ├── processed
│   └── raw
├── dockerfiles/              # Dockerfiles
│   ├── api.Dockerfile
│   └── train.Dockerfile
├── docs/                     # Documentation
│   ├── mkdocs.yml
│   └── source/
│       └── index.md
├── models/                   # Trained models
├── notebooks/                # Jupyter notebooks
├── reports/                  # Reports
│   └── figures/
├── src/                      # Source code
│   ├── project_name/
│   │   ├── __init__.py
│   │   ├── api.py
│   │   ├── data.py
│   │   ├── evaluate.py
│   │   ├── models.py
│   │   ├── train.py
│   │   └── visualize.py
└── tests/                    # Tests
│   ├── __init__.py
│   ├── test_api.py
│   ├── test_data.py
│   └── test_model.py
├── .gitignore
├── .pre-commit-config.yaml
├── LICENSE
├── pyproject.toml            # Python project file
├── README.md                 # Project README
├── requirements.txt          # Project requirements
├── requirements_dev.txt      # Development requirements
└── tasks.py                  # Project tasks
```


Created using [mlops_template](https://github.com/SkafteNicki/mlops_template),
a [cookiecutter template](https://github.com/cookiecutter/cookiecutter) for getting
started with Machine Learning Operations (MLOps).
