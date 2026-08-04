# ai-test-generator-gitops

> GitOps deployment-state repository for [ai-test-generator](https://github.com/SannidhiSriram-06/ai-test-generator). This repo holds nothing but the Helm chart and its config — it's the single source of truth ArgoCD watches to deploy the app to Kubernetes. No application code lives here.

---

## Why this repo exists

Separating deployment state from application code is the core GitOps principle this project demonstrates: the app repo's CI never talks to Kubernetes directly. It only updates `values.yaml` here. ArgoCD is the only thing that ever applies changes to the cluster, and it does so by continuously reconciling against whatever is committed to this repo.

```
ai-test-generator (app repo)
   │  Jenkins CI: test → build → push image to ECR
   ▼
ai-test-generator-gitops (this repo)
   │  Jenkins CI updates charts/ai-test-generator/values.yaml (image.tag)
   ▼
ArgoCD
   │  detects the change, runs `helm upgrade`
   ▼
Kubernetes (Minikube)
   │  new pod rolled out with the new image
```

---

## Contents

```
ai-test-generator-gitops/
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
| `env.GROQ_API_KEY` | placeholder — actual key is injected via a Kubernetes Secret (`ai-test-generator-secret`), not this file |

`templates/deployment.yaml` pulls images from a private ECR repo via an `ecr-secret` image pull secret, and reads the Groq API key from a `secretKeyRef`, not from plaintext values.

---

## How a deploy happens

1. A commit lands on `main` in the **app repo**.
2. Jenkins there runs tests (70% coverage gate), builds a Docker image tagged `BUILD_NUMBER-COMMIT_SHA`, and pushes it to AWS ECR.
3. Jenkins clones **this repo**, updates `image.tag` in `charts/ai-test-generator/values.yaml`, commits, and pushes.
4. ArgoCD, watching this repo, detects the diff and runs `helm upgrade` against the Minikube cluster.
5. The new pod comes up running the freshly built image — no manual `kubectl` or `helm` commands involved.

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

This repo intentionally contains no application source, Dockerfile, or CI logic — that all lives in [ai-test-generator](https://github.com/SannidhiSriram-06/ai-test-generator). Keeping them separate is what allows ArgoCD to reconcile deployment state independently of application build state, which is the point of the GitOps split.

---

## License

MIT
