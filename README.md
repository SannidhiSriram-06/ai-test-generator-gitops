# AI Test Generator

> FastAPI service that uses Groq LLM to automatically generate pytest test cases from Python source code, deployed via a full GitOps CI/CD pipeline.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Developer Laptop                          │
│                                                                  │
│   git push                                                       │
│      │                                                           │
│      ▼                                                           │
│  ┌──────────┐    poll    ┌──────────────────────────────┐       │
│  │  GitHub   │ ─────────▶│       Jenkins Pipeline        │       │
│  │ (app repo)│           │                              │       │
│  └──────────┘           │  1. pytest (≥70% cov gate)   │       │
│                          │  2. docker build             │       │
│                          │  3. docker push → ECR        │       │
│                          │  4. update values.yaml       │       │
│                          └──────────────┬───────────────┘       │
│                                         │ git push               │
│                                         ▼                        │
│                          ┌──────────────────────────────┐       │
│                          │   GitHub (gitops repo)        │       │
│                          │   charts/ai-test-generator/   │       │
│                          │   values.yaml  ← image tag    │       │
│                          └──────────────┬───────────────┘       │
│                                         │ auto-sync              │
│                                         ▼                        │
│                          ┌──────────────────────────────┐       │
│                          │          ArgoCD               │       │
│                          │  (watches gitops repo)        │       │
│                          └──────────────┬───────────────┘       │
│                                         │ helm upgrade           │
│                                         ▼                        │
│                          ┌──────────────────────────────┐       │
│                          │      Minikube Cluster         │       │
│                          │  ┌────────────────────────┐  │       │
│                          │  │    Helm Release         │  │       │
│                          │  │   ai-test-generator     │  │       │
│                          │  │   FastAPI pod (ECR img) │  │       │
│                          │  └────────────────────────┘  │       │
│                          └──────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white)
![Python](https://img.shields.io/badge/Python_3.11-3776AB?style=flat&logo=python&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat&logo=jenkins&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=flat&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)
![AWS ECR](https://img.shields.io/badge/AWS_ECR-FF9900?style=flat&logo=amazon-aws&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)
![Groq](https://img.shields.io/badge/Groq_LLM-F55036?style=flat&logoColor=white)

---

## How it works

Send a Python function to the `/generate` endpoint and the service forwards it to Groq's `llama-3.1-8b-instant` model with a structured prompt instructing it to return a complete pytest test suite. The API enforces a rate limit of 5 requests per minute via `slowapi` and validates that input code is non-empty and under the size limit before hitting the LLM. Every push to `main` triggers a Jenkins pipeline that runs pytest with a 70% coverage gate, builds a new Docker image tagged `BUILD_NUMBER-COMMIT_SHA`, pushes it to AWS ECR, and updates the image tag in the GitOps repository — which ArgoCD detects and uses to roll out a new pod on Minikube automatically.

---

## Folder structure

```
ai-test-generator/
├── app/
│   ├── __init__.py
│   ├── config.py          # env var loading via python-dotenv
│   └── main.py            # FastAPI app, /health and /generate endpoints
├── tests/
│   ├── __init__.py
│   └── test_main.py       # 5 async pytest tests, 87% coverage
├── Terraform/
│   └── main.tf            # ECR repo provisioning (ap-south-1)
├── Dockerfile             # multi-stage build, python:3.11-slim
├── Jenkinsfile            # CI pipeline (pytest → ECR → gitops update)
├── pytest.ini             # asyncio_mode = auto
├── requirements.txt
└── README.md
```

---

## Local setup

### Prerequisites

- Python 3.11+
- Docker Desktop
- A [Groq API key](https://console.groq.com/)

### Steps

```bash
# 1. Clone the repo
git clone https://github.com/SannidhiSriram-06/ai-test-generator.git
cd ai-test-generator

# 2. Create a virtual environment
python3 -m venv venv
source venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set up environment variables
echo "GROQ_API_KEY=your_key_here" > .env

# 5. Start the server
uvicorn app.main:app --reload --port 8000
```

The API is now running at `http://localhost:8000`.

### Usage

```bash
# Health check
curl http://localhost:8000/health

# Generate tests
curl -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d '{"code": "def add(a, b):\n    return a + b"}'
```

---

## Running tests

```bash
# Run tests with coverage report
pytest tests/ --cov=app --cov-report=term-missing -v

# Run with the same coverage gate used in CI
pytest tests/ --cov=app --cov-fail-under=70 -v
```

Expected output: **5 passed, 87% coverage**.

---

## CI/CD flow

Every push to `main` triggers the Jenkins pipeline defined in `Jenkinsfile`:

| Stage | What happens |
|---|---|
| **Checkout** | Jenkins pulls the latest commit from GitHub |
| **Install dependencies** | `pip install` into a virtualenv |
| **Run tests** | `pytest` with `--cov-fail-under=70` — build fails if coverage drops below threshold |
| **Build & push to ECR** | Docker image built, tagged `BUILD_NUMBER-COMMIT_SHA`, pushed to AWS ECR (`ap-south-1`) |
| **Update GitOps repo** | Jenkins clones `ai-test-generator-gitops`, updates `image.tag` in `values.yaml`, and pushes |
| **ArgoCD sync** | ArgoCD detects the `values.yaml` change and rolls out the new image to Minikube automatically |

Secrets (AWS credentials, Groq API key, GitHub PAT) are stored in the Jenkins credential store — never in code.

---

## Screenshots

> See `project-screenshots.pdf` in this repository for full pipeline screenshots including Jenkins build view, ArgoCD sync status, and kubectl pod output.

---

## License

MIT
