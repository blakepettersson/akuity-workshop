# Roche Workshop — Kargo + Argo CD on Akuity Platform

Promote a simple NGINX app through `dev → test → prod` using Kargo and Argo CD on the Akuity Platform.

## Prerequisites

- Akuity account and access to the Argo CD and Kargo instances
- A Kubernetes cluster registered to Argo CD (an Argo CD agent)
- A Kargo agent connected to your Kargo instance
- A fork of this repo in your own GitHub account
- A GitHub personal access token (PAT) with read and write access to your fork
- CLIs: `akuity`, `kargo`, [`task`](https://taskfile.dev/docs/installation), `envsubst` (`brew install akuity kargo go-task gettext`)

## Step-by-Step Instructions

### 1. Clone your fork and set up `.env`

```bash
git clone https://github.com/<your-github-username>/roche-workshop.git
cd roche-workshop
cp .env.example .env
```

Fill in `.env`. `WORKSHOP_NAME` must be **unique per participant** (for example, `workshop-shivam`). It's used as your Kargo Project name and as the prefix for your Argo CD Applications.

> Never commit `.env`. It contains your PAT.

### 2. Log in and check your config

```bash
akuity login
task check
```

### 3. Create the Argo CD AppProject

```bash
task apply-argocd-project
```

This creates the shared `akuity-workshop` AppProject.

### 4. Create the Argo CD Applications

```bash
task apply-applicationset
```

In the Argo CD dashboard you should see three Applications:

`<WORKSHOP_NAME>-nginx-dev`, `<WORKSHOP_NAME>-nginx-test`, `<WORKSHOP_NAME>-nginx-prod`

They show as **Unknown** and aren't synced. That's expected: each one points at a `stage/<env>` branch that doesn't exist yet. Kargo creates those branches when you promote.

### 5. Create the Kargo resources

```bash
task apply-kargo
```

This applies, in order:

| Resource | What it does |
|---|---|
| Project | Your own Kargo Project, named `<WORKSHOP_NAME>` |
| Secret | Git credentials so Kargo can clone your fork, create `stage/*` branches, and push commits |
| Warehouse | Watches `public.ecr.aws/nginx/nginx` for new tags (`^1.27.0`) |
| PromotionTask | Defines how Freight is promoted: clone, set the image, `kustomize build`, commit, push, and sync Argo CD |
| Stages | `dev → test → prod`. Each stage can only promote Freight that passed the stage before it |

The pipeline is ready. Freight discovered by the Warehouse can now be promoted through the stages.

> Tip: `task setup` runs steps 2–5 in one go.

### 6. Your first promotion

1. **dev**: in the Kargo UI, drag the Freight onto the `dev` stage.
2. **test**: in the Kargo UI, click the truck icon on the `test` stage and pick the Freight.
3. **prod**: use the CLI:

   ```bash
   kargo login https://<your-kargo-instance-url> --sso
   kargo get freight --project <WORKSHOP_NAME>
   kargo promote --project <WORKSHOP_NAME> --freight <freight-name> --stage prod
   ```

After each promotion, Kargo pushes to `stage/<env>` and syncs the matching Argo CD Application.

## Repository Structure

```text
.
├── app/                 # NGINX Kustomize base + dev/test/prod overlays
├── argocd/              # AppProject, ApplicationSet
├── kargo/               # Project, Warehouse, PromotionTask, Stages
├── secret.yaml          # Kargo Git credentials (filled from .env)
├── Taskfile.yaml
└── .env.example
```
