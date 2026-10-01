# Akuity Workshop — Kargo + Argo CD on Akuity Platform

Promote a simple NGINX app through `dev → test → prod` using Kargo and Argo CD on the Akuity Platform.

## Prerequisites

- Akuity account (https://akuity.cloud) and access to the Argo CD and Kargo instances
- [Kind Cluster](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- Access to both Argo CD and Kargo Instance control planes
- A fork of this repo in your own GitHub account
- A GitHub personal access token (PAT) with read and write access to your fork
- CLIs: [`akuity`](https://docs.akuity.io/akuity-portal/automation/#installation), [`kargo`](https://docs.akuity.io/kargo/getting-started/access-kargo-instance#access-kargo-using-the-kargo-cli), [`task`](https://taskfile.dev/docs/installation), [`envsubst`](https://formulae.brew.sh/formula/gettext)
(`brew install akuity kargo go-task gettext`)

## Step-by-Step Instructions

### 1. Create your cluster and connect the agents

```bash
kind create cluster --name <WORKSHOP_NAME>
```

1. **Argo CD agent**: register this cluster with your Argo CD instance's control plane. Follow [Connect a Kubernetes cluster](https://docs.akuity.io/argocd/getting-started/connect-kubernetes-cluster). Note the cluster name you choose: it's your `ARGOCD_DESTINATION` in `.env`.
2. **Kargo agent**: create a Kargo agent for your Kargo instance and install it on the same cluster. Follow [Connect a Kargo agent](https://docs.akuity.io/kargo/getting-started/connect-kargo-agent).

Wait until both agents show as **Healthy** in the Akuity Platform UI.

### 2. Clone your fork and set up `.env`

```bash
git clone https://github.com/<your-github-username>/akuity-workshop.git
cd akuity-workshop
cp .env.example .env
```

Fill in `.env`. `WORKSHOP_NAME` must be **unique per participant** (for example, `workshop-shivam`). It's used as your Kargo Project name and as the prefix for your Argo CD Applications.

> Never commit `.env`. It contains your PAT.

### 3. Log in and check your config

```bash
akuity login
task check
```

### 4. Create the Argo CD AppProject

```bash
task apply-argocd-project
```

This creates the shared `akuity-workshop` AppProject.

### 5. Create the Argo CD Applications

```bash
task apply-applicationset
```

In the Argo CD dashboard you should see three Applications:

`<WORKSHOP_NAME>-nginx-dev`, `<WORKSHOP_NAME>-nginx-test`, `<WORKSHOP_NAME>-nginx-prod`

They show as **Unknown** and aren't synced. That's expected: each one points at a `stage/<env>` branch that doesn't exist yet. Kargo creates those branches when you promote.

### 6. Create the Kargo resources

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

> Tip: `task setup` runs steps 3–6 in one go.

### 7. Your first promotion

1. **dev**: in the Kargo UI, drag the Freight onto the `dev` stage.
2. **test**: in the Kargo UI, click the truck icon on the `test` stage and pick the Freight.
3. **prod**: use the CLI:

   ```bash
   kargo login https://<your-kargo-instance-url> --sso
   kargo get freight --project <WORKSHOP_NAME>
   kargo promote --project <WORKSHOP_NAME> --freight <freight-name> --stage prod
   ```

After each promotion, Kargo pushes to `stage/<env>` and syncs the matching Argo CD Application.

### 8. Check the app

Each stage runs in its own namespace, `<WORKSHOP_NAME>-<stage>`. Port-forward to a stage's Service:

```bash
task port-forward STAGE=dev
```

Or with `kubectl` directly:

```bash
kubectl port-forward svc/nginx 8080:80 -n <WORKSHOP_NAME>-dev
```

Open http://localhost:8080. You should see **NGINX - DEV**.

Check which image is running:

```bash
kubectl get deploy nginx -n <WORKSHOP_NAME>-dev \
-o jsonpath='{.spec.template.spec.containers[0].image}'
```

The tag should match the Freight you promoted. Repeat for `test` and `prod`, using a different local port for each, for example `task port-forward STAGE=test PORT=8081`.


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
