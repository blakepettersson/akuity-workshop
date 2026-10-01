# Roche Workshop — Kargo + Argo CD on Akuity Platform

This workshop demonstrates a simple GitOps deployment flow using **GitHub → Kargo → Argo CD → Kubernetes**.

The application used in the workshop is a simple NGINX deployment promoted through three environments:

```text
GitHub main
     ↓
Kargo Warehouse
     ↓
Kargo Stages
dev → test → prod
     ↓
stage/<env> Git branches
     ↓
Argo CD
     ↓
Kubernetes
```

Each workshop participant gets their own Kargo Project and destination cluster.

## Prerequisites

Make sure the following tools are installed before starting.

| Tool | Purpose | Installation |
|---|---|---|
| [Akuity CLI](https://docs.akuity.io/akuity-portal/automation/) | Manage Akuity Platform resources | `brew install akuity` |
| [Kargo CLI](https://docs.kargo.io/user-guide/cli/installation) | Manage Kargo resources and promotions | `brew install kargo` |
| [Argo CD CLI](https://argo-cd.readthedocs.io/en/latest/user-guide/commands/argocd/) | Interact with Argo CD | `brew install argocd` |
| [Task](https://taskfile.dev/docs/installation) | Run workshop automation tasks | See installation guide |
| [Git](https://git-scm.com/downloads) | A fork of this repo, inside of your own repo | See installation guide |
| `envsubst` | Substitute environment variables in YAML | `brew install gettext` |

### macOS

Install `envsubst`:

```bash
brew install gettext
echo 'export PATH="/opt/homebrew/opt/gettext/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Verify the tools:

```bash
akuity version
kargo version
argocd version --client
task --version
git --version
envsubst --version
```

## 1. Clone the Repository

Clone your workshop repository:

```bash
git clone https://github.com/<your-github-username>/roche-workshop.git
cd roche-workshop
```

## 2. Configure Environment Variables

Copy the example environment file:

```bash
cp .env.example .env
```

Update `.env` with your values:

```bash
AKUITY_ARGOCD_INSTANCE=<your-argocd-instance>
AKUITY_KARGO_INSTANCE=<your-kargo-instance>
AKUITY_ORG_NAME=akuity

GITOPS_REPO_URL=https://github.com/<your-github-username>/roche-workshop.git

GITHUB_USERNAME=<your-github-username>
GITHUB_PAT=<your-github-pat>

ARGOCD_DESTINATION=<your-destination-cluster>

WORKSHOP_NAME=<your-unique-workshop-name>
```

### Important

`WORKSHOP_NAME` must be **unique for each participant**.

For example:

```bash
WORKSHOP_NAME=workshop-shivam
```

This value is used for:

- Kargo Project
- Kargo Stages
- Kargo PromotionTask
- Argo CD ApplicationSet
- Argo CD Applications
- Kubernetes namespaces
- Kargo → Argo CD authorization

The Argo CD `AppProject` remains shared:

```text
roche-workshop
```

For example, with:

```bash
WORKSHOP_NAME=workshop-shivam
```

the generated resources will be:

```text
Kargo Project:
  workshop-shivam

Argo CD ApplicationSet:
  workshop-shivam

Argo CD Applications:
  workshop-shivam-nginx-dev
  workshop-shivam-nginx-test
  workshop-shivam-nginx-prod

Namespaces:
  workshop-shivam-dev
  workshop-shivam-test
  workshop-shivam-prod
```

## 3. Authenticate with Akuity

Log in to Akuity Platform:

```bash
akuity login
```

Verify your identity:

```bash
akuity whoami
```

## 4. Verify Configuration

Run:

```bash
task check
```

This validates that all required environment variables are configured.

## 5. Configure GitHub Access

The workshop uses a GitHub Personal Access Token to allow Kargo to clone and push to the GitOps repository.

Set the credentials in `.env`:

```bash
GITHUB_USERNAME=<your-github-username>
GITHUB_PAT=<your-github-pat>
```

> **Security:** Never commit `.env` or your GitHub token to the repository.

## 6. Create the Argo CD AppProject

The workshop uses a shared Argo CD AppProject:

```text
roche-workshop
```

Apply it with:

```bash
task apply-argocd-project
```

## 7. Create the Kargo Resources

Create the participant-specific Kargo Project, Warehouse, Stages, credentials, and PromotionTask:

```bash
task apply-kargo
```

The resulting Kargo structure is:

```text
<WORKSHOP_NAME>
├── Warehouse
│   └── nginx
│
├── Stage
│   ├── dev
│   ├── test
│   └── prod
│
└── PromotionTask
    └── nginx-promotion
```

## 8. Create the Argo CD ApplicationSet

Apply the ApplicationSet:

```bash
task apply-applicationset
```

The ApplicationSet creates one Argo CD Application for each environment:

```text
<WORKSHOP_NAME>-nginx-dev
<WORKSHOP_NAME>-nginx-test
<WORKSHOP_NAME>-nginx-prod
```

Each Application points to the corresponding Kargo-managed Git branch:

```text
stage/dev
stage/test
stage/prod
```

## 9. Application Deployment Flow

The NGINX application is deployed through three environments.

### Development

```text
Kargo dev
   ↓
stage/dev
   ↓
Argo CD
   ↓
Kubernetes
```

### Test

```text
Kargo test
   ↓
stage/test
   ↓
Argo CD
   ↓
Kubernetes
```

### Production

```text
Kargo prod
   ↓
stage/prod
   ↓
Argo CD
   ↓
Kubernetes
```

## 10. Promote an Image

List available Freight:

```bash
kargo get freight --project <your-workshop-name>
```

Promote Freight to `dev`:

```bash
kargo promote \
  --project <your-workshop-name> \
  --stage dev \
  --freight-alias <freight-alias>
```

After the promotion completes, Kargo will:

1. Clone the `main` branch.
2. Clone/create the `stage/dev` branch.
3. Update the NGINX image.
4. Build the Kustomize overlay.
5. Commit the rendered manifests.
6. Push the changes to GitHub.
7. Update the Argo CD Application.
8. Argo CD synchronizes the application to Kubernetes.

## 11. Promote Through the Environments

After validating `dev`, promote the same Freight to `test`:

```bash
kargo promote \
  --project <your-workshop-name> \
  --stage test \
  --freight-alias <freight-alias>
```

Then promote it to `prod`:

```bash
kargo promote \
  --project <your-workshop-name> \
  --stage prod \
  --freight-alias <freight-alias>
```

The same artifact therefore moves through:

```text
        ┌─────────┐
        │ GitHub  │
        │  main   │
        └────┬────┘
             │
             ▼
      ┌─────────────┐
      │    Kargo    │
      │  Warehouse  │
      └──────┬──────┘
             │
             ▼
          ┌─────┐
          │ dev │
          └──┬──┘
             │
             ▼
          ┌──────┐
          │ test │
          └──┬───┘
             │
             ▼
          ┌──────┐
          │ prod │
          └──────┘
             │
             ▼
        ┌──────────┐
        │  Argo CD │
        └────┬─────┘
             │
             ▼
        ┌──────────┐
        │Kubernetes│
        └──────────┘
```

## 12. Verify in Argo CD

List the applications:

```bash
argocd app list
```

You should see:

```text
<WORKSHOP_NAME>-nginx-dev
<WORKSHOP_NAME>-nginx-test
<WORKSHOP_NAME>-nginx-prod
```

Check an application:

```bash
argocd app get <WORKSHOP_NAME>-nginx-dev
```

You can also inspect the applications from the Akuity Platform UI.

## 13. Verify in Kubernetes

The destination cluster is configured through:

```bash
ARGOCD_DESTINATION=<your-destination-cluster>
```

The application is deployed into:

```text
<WORKSHOP_NAME>-dev
<WORKSHOP_NAME>-test
<WORKSHOP_NAME>-prod
```

For example:

```bash
kubectl get pods -n <WORKSHOP_NAME>-dev
```

Check the Service:

```bash
kubectl get svc -n <WORKSHOP_NAME>-dev
```

The NGINX Service uses `NodePort`, but Kubernetes dynamically allocates the NodePort. This avoids hardcoded port collisions between deployments.

## 14. Repository Structure

```text
.
├── app/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   │
│   └── overlays/
│       ├── dev/
│       │   ├── configmap.yaml
│       │   ├── kustomization.yaml
│       │   └── namespace.yaml
│       │
│       ├── test/
│       │   ├── configmap.yaml
│       │   ├── kustomization.yaml
│       │   └── namespace.yaml
│       │
│       └── prod/
│           ├── configmap.yaml
│           ├── kustomization.yaml
│           └── namespace.yaml
│
├── argocd/
│   ├── app-project.yaml
│   ├── applicationset.yaml
│   └── kargo-application.yaml
│
├── kargo/
│   ├── project.yaml
│   ├── secret.yaml
│   ├── warehouse.yaml
│   ├── promotion-task.yaml
│   └── stages.yaml
│
├── taskfile.yaml
├── .env.example
└── README.md
```

## 15. Useful Commands

### Kargo

List Projects:

```bash
kargo get projects
```

List Warehouses:

```bash
kargo get warehouses --project <your-workshop-name>
```

List Stages:

```bash
kargo get stages --project <your-workshop-name>
```

List Freight:

```bash
kargo get freight --project <your-workshop-name>
```

List Promotions:

```bash
kargo get promotions --project <your-workshop-name>
```

### Argo CD

List applications:

```bash
argocd app list
```

Get application status:

```bash
argocd app get <WORKSHOP_NAME>-nginx-dev
```

View application history:

```bash
argocd app history <WORKSHOP_NAME>-nginx-dev
```

### Kubernetes

```bash
kubectl get pods -A
kubectl get applications -n argocd
kubectl get svc -n <WORKSHOP_NAME>-dev
```

## 16. Cleanup

Remove the workshop resources when finished:

```bash
task delete
```

Or remove the participant-specific Kargo Project and Argo CD resources manually.

Because each participant uses a unique `WORKSHOP_NAME`, cleanup only affects that participant's resources.

## What This Workshop Demonstrates

By the end of the workshop, you will have seen:

- **GitHub** as the GitOps source
- **Kargo Warehouse** discovering new application versions
- **Kargo Freight** representing promotable artifacts
- **Kargo Stages** representing `dev → test → prod`
- **Kargo PromotionTasks** updating Git manifests
- **Git branches** representing environment state
- **Kustomize** rendering environment-specific manifests
- **Argo CD ApplicationSet** creating environment Applications
- **Argo CD** synchronizing Git state to Kubernetes
- **Akuity Platform** providing the managed Argo CD/Kargo environment

The complete flow is:

```text
New NGINX image
      ↓
Kargo Warehouse
      ↓
Freight
      ↓
Kargo dev
      ↓
Git: stage/dev
      ↓
Argo CD
      ↓
Kubernetes

      ↓ promote same Freight

Kargo test
      ↓
Git: stage/test
      ↓
Argo CD
      ↓
Kubernetes

      ↓ promote same Freight

Kargo prod
      ↓
Git: stage/prod
      ↓
Argo CD
      ↓
Kubernetes
```
