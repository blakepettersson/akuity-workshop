# Kargo + Argo CD Workshop

This workshop demonstrates a simple GitOps deployment flow using **Akuity Platform, Kargo, Argo CD, GitHub, and Kubernetes**.

You will deploy an NGINX application through three environments:

```text
GitHub
   │
   ▼
Kargo Warehouse
   │
   ▼
dev → test → prod
   │
   ▼
Argo CD
   │
   ▼
Kubernetes
```

Kargo is responsible for promoting container image versions between environments, while Argo CD continuously manages the Kubernetes resources.

---

# Prerequisites

Before starting, make sure you have:

- An Akuity account: https://training.akuity.cloud
- `akuity` CLI installed
- `task` installed
- `envsubst` available
- A Kubernetes cluster created with `kind`
- An Argo CD Instance
- A Kargo Instance
- Access to the Argo CD and Kargo control planes
- An Argo CD Agent registered with the control plane
- A Kargo Agent registered with the control plane
- A fork of this repository in your own GitHub account
- A GitHub Personal Access Token (PAT) with read/write access to your fork

---

# 1. Register your agents

First, create an **Argo CD Agent** and register your Kubernetes cluster with the Akuity Platform.

Then create a **Kargo Agent** and register it with the Kargo control plane.

You should have:

```text
Akuity Platform
├── Argo CD Instance
│   └── Argo CD Agent → your Kubernetes cluster
│
└── Kargo Instance
    └── Kargo Agent
```

---

# 2. Login to Akuity

Run:

```bash
akuity login
```

---

# 3. Configure your environment

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env` and provide your values:

```bash
AKUITY_ARGOCD_INSTANCE=<your-argocd-instance>
AKUITY_KARGO_INSTANCE=<your-kargo-instance>
AKUITY_ORG_NAME=akuity

GITOPS_REPO_URL=https://github.com/<your-github-username>/roche-workshop.git

GITHUB_USERNAME=<your-github-username>
GITHUB_PAT=<your-github-pat>

ARGOCD_DESTINATION=roche-workshop
WORKSHOP_NAME=roche-workshop
```

The GitHub PAT is used by Kargo to access your GitOps repository.

Make sure `.env` is **not committed to Git**.

---

# 4. Validate your configuration

Run:

```bash
task check
```

You should see:

```text
✓ All required variables are set
```

If a variable is missing, the task will tell you which one needs to be configured.

---

# 5. Create the Argo CD AppProject

The workshop uses a shared Argo CD AppProject.

Run:

```bash
task apply-argocd-project
```

This creates the:

```text
roche-workshop
```

AppProject in Argo CD.

---

# 6. Create the Argo CD Applications

The ApplicationSet creates three Argo CD Applications:

```text
roche-workshop-nginx-dev
roche-workshop-nginx-test
roche-workshop-nginx-prod
```

Run:

```bash
task apply-applicationset
```

The Taskfile automatically substitutes the values from `.env` before applying the manifest.

For example:

```yaml
repoURL: ${GITOPS_REPO_URL}
```

is replaced with your actual repository URL.

The resulting Applications track:

```text
stage/dev
stage/test
stage/prod
```

in your Git repository.

At this point, the stage branches may not exist yet, so the Applications can show an `Unknown` or unhealthy state. This is expected.

Kargo will create and update these branches during promotion.

---

# 7. Create the Kargo resources

The workshop uses the following Kargo resources:

```text
Kargo Project
    │
    ├── Warehouse
    │
    ├── PromotionTask
    │
    └── Stages
          ├── dev
          ├── test
          └── prod
```

You can create all of them with:

```bash
task apply-kargo
```

This runs:

```bash
task apply-kargo-project
task apply-kargo-secret
task apply-kargo-warehouse
task apply-kargo-promotion-task
task apply-kargo-stages
```

## Kargo Project

The Kargo Project is created using:

```bash
task apply-kargo-project
```

The project name comes from:

```bash
WORKSHOP_NAME=roche-workshop
```

---

# 8. Configure GitHub credentials

Kargo needs credentials to access your GitOps repository.

These credentials allow Kargo to:

- Read the source configuration from the repository
- Create environment-specific branches
- Update those branches during promotion
- Commit rendered Kubernetes manifests
- Push changes back to GitHub

The credentials are configured with:

```bash
task apply-kargo-secret
```

The GitHub credentials come from your `.env`:

```bash
GITOPS_REPO_URL=<your repository>
GITHUB_USERNAME=<your username>
GITHUB_PAT=<your PAT>
```

The PAT must have permission to read from and write to the repository.

---

# 9. Create the Warehouse

The Warehouse discovers new versions of the NGINX container image.

Run:

```bash
task apply-kargo-warehouse
```

The Warehouse watches:

```text
public.ecr.aws/nginx/nginx
```

for versions matching:

```text
^1.27.0
```

Once Kargo discovers a new image version, it creates Freight that can be promoted through the pipeline.

---

# 10. Create the PromotionTask

The PromotionTask defines what happens when Freight is promoted.

Run:

```bash
task apply-kargo-promotion-task
```

The PromotionTask performs the following:

1. Clones the `main` branch.
2. Checks out the target `stage/<environment>` branch.
3. Updates the NGINX image version.
4. Builds the Kustomize manifests.
5. Writes the rendered manifests to the stage branch.
6. Commits the changes.
7. Pushes the branch to GitHub.
8. Updates the corresponding Argo CD Application to the resulting commit.

The flow is:

```text
main
 │
 │ Kargo promotion
 ▼
Kustomize render
 │
 ▼
stage/dev
stage/test
stage/prod
 │
 ▼
Argo CD
```

---

# 11. Create the deployment stages

Run:

```bash
task apply-kargo-stages
```

This creates:

```text
dev → test → prod
```

The stages are configured so that:

- `dev` receives Freight directly from the Warehouse.
- `test` receives Freight promoted from `dev`.
- `prod` receives Freight promoted from `test`.

The pipeline is now ready.

---

# 12. Verify the setup

In the Kargo dashboard, you should see:

```text
roche-workshop
└── nginx
    ├── dev
    ├── test
    └── prod
```

In the Argo CD dashboard, you should see:

```text
roche-workshop-nginx-dev
roche-workshop-nginx-test
roche-workshop-nginx-prod
```

The Applications may initially show an `Unknown` or unhealthy state because the `stage/*` branches do not exist yet.

These branches are created by Kargo during the first promotion.

---

# 13. Your first promotion

Once Freight has been discovered by the Warehouse, promote it through the pipeline.

## Promote to dev

In the Kargo dashboard:

1. Open the `dev` Stage.
2. Select the available Freight.
3. Promote the Freight to `dev`.

Kargo will:

- Create/update `stage/dev`
- Render the Kubernetes manifests
- Commit them to GitHub
- Push the branch
- Update the Argo CD Application

Argo CD will then synchronize the rendered manifests to:

```text
roche-workshop-dev
```

---

## Promote to test

Once `dev` is healthy:

1. Open the `test` Stage in Kargo.
2. Select the Freight from `dev`.
3. Promote it to `test`.

Kargo creates/updates:

```text
stage/test
```

and Argo CD deploys the rendered manifests to:

```text
roche-workshop-test
```

---

## Promote to prod using the CLI

You can also promote Freight using the Kargo CLI.

First authenticate to your Kargo instance as required by your environment.

Then:

```bash
kargo promote \
  --project roche-workshop \
  --stage prod \
  --freight-alias <freight-alias>
```

This promotes the selected Freight to:

```text
prod
```

Kargo updates:

```text
stage/prod
```

and Argo CD deploys the resulting manifests to:

```text
roche-workshop-prod
```

---

# 14. Final GitOps flow

After completing the workshop, the complete flow looks like this:

```text
                    GitHub
                      │
                      │
              ┌───────▼───────┐
              │    Warehouse  │
              │    NGINX      │
              └───────┬───────┘
                      │
                    Freight
                      │
                      ▼
                 ┌─────────┐
                 │   dev   │
                 └────┬────┘
                      │
              stage/dev branch
                      │
                      ▼
                 ┌─────────┐
                 │  ArgoCD │
                 └────┬────┘
                      │
                      ▼
              Kubernetes / dev
                      │
                      │
                   promote
                      │
                      ▼
                 ┌─────────┐
                 │  test   │
                 └────┬────┘
                      │
              stage/test branch
                      │
                      ▼
                 ┌─────────┐
                 │  ArgoCD │
                 └────┬────┘
                      │
                      ▼
              Kubernetes / test
                      │
                      │
                   promote
                      │
                      ▼
                 ┌─────────┐
                 │  prod   │
                 └────┬────┘
                      │
              stage/prod branch
                      │
                      ▼
                 ┌─────────┐
                 │  ArgoCD │
                 └────┬────┘
                      │
                      ▼
              Kubernetes / prod
```

The key idea is:

```text
Kargo controls promotion.
GitHub stores the desired manifests.
Argo CD deploys the manifests.
Kubernetes runs the application.
```

---

# Useful Taskfile commands

Check configuration:

```bash
task check
```

Apply the shared Argo CD AppProject:

```bash
task apply-argocd-project
```

Apply the ApplicationSet:

```bash
task apply-applicationset
```

Apply individual Kargo resources:

```bash
task apply-kargo-project
task apply-kargo-secret
task apply-kargo-warehouse
task apply-kargo-promotion-task
task apply-kargo-stages
```

Apply all Kargo resources:

```bash
task apply-kargo
```

Apply the Argo CD Application that manages Kargo resources:

```bash
task apply-kargo-application
```

Set up the workshop:

```bash
task setup
```