# ai-test-generator-gitops

> Helm-chart repository for [ai-test-generator](https://github.com/SannidhiSriram-06/ai-test-generator). It contains deployment configuration only; no application source code, Dockerfile, or CI pipeline is included. An Argo CD Application configured outside this repository can use this chart as its desired deployment state.

---

## Why this repo exists

Separating deployment configuration from application code supports a GitOps workflow. The app repository's Jenkinsfile updates `values.yaml` here and contains no Kubernetes commands. Argo CD and cluster configuration are external prerequisites, so this repository alone does not define or prove the running Argo CD application.

```
ai-test-generator (app repo)
   │  Jenkins CI: test → build → push image to ECR
   ▼
ai-test-generator-gitops (this repo)
   │  Jenkins CI updates charts/ai-test-generator/values.yaml (image.tag)
   ▼
ArgoCD
   │  renders the Helm chart and syncs the manifests
   ▼
Kubernetes (Minikube)
   │  new pod rolled out with the new image
```

---

## Contents

```
ai-pytest-generator-gitops/
└── charts/
    └── ai-test-generator/
        ├── Chart.yaml          # Helm chart metadata
        ├── values.yaml         # image tag, replica count, resources, service config
        └── templates/
            ├── deployment.yaml
            └── service.yaml
```

---

## Chart configuration

`charts/ai-test-generator/values.yaml` controls the release:

| Key | Purpose |
|---|---|
| `image.repository` / `image.tag` | ECR image to deploy — `image.tag` is the field Jenkins rewrites on every CI run |
| `replicaCount` | number of pods |
| `service.type`, `service.port`, `service.nodePort` | exposed as a `NodePort` service on Minikube |
| `resources.requests` / `resources.limits` | CPU/memory (256Mi/250m requests, 512Mi/500m limits) |
| `env.GROQ_API_KEY` | placeholder only; the deployment template does not use it. The actual key is read from `ai-test-generator-secret` |

`templates/deployment.yaml` pulls images from a private ECR repo via an `ecr-secret` image pull secret, and reads the Groq API key from a `secretKeyRef`, not from plaintext values.

---

## How a deploy happens

1. A commit lands on `main` in the **app repo**.
2. Jenkins there runs tests (70% coverage gate), builds a Docker image tagged `BUILD_NUMBER-COMMIT_SHA`, and pushes it to AWS ECR.
3. Jenkins clones **this repo**, updates `image.tag` in `charts/ai-test-generator/values.yaml`, commits, and pushes.
4. If an external Argo CD Application watches this repository, it can detect the diff, render the chart, and sync the resulting manifests to its configured cluster.
5. The Deployment references the newly selected image tag. A running workload also requires the pre-existing `ecr-secret` image-pull secret and `ai-test-generator-secret` application secret.

---

## Local verification

To render or install the chart manually (e.g. for debugging without ArgoCD):

```bash
git clone https://github.com/SannidhiSriram-06/ai-pytest-generator-gitops.git
cd ai-pytest-generator-gitops

helm template ai-test-generator charts/ai-test-generator
# or, against a running cluster:
helm upgrade --install ai-test-generator charts/ai-test-generator
```

---

## Relationship to the app repo

This repo intentionally contains no application source, Dockerfile, or CI logic — that all lives in [ai-test-generator](https://github.com/SannidhiSriram-06/ai-test-generator). The separate chart can be reconciled independently when an Argo CD Application is configured to watch it.

---

## License

No license file is currently included in this repository.
