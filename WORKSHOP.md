# Workshop: Build your own simple Helm hydration CI/CD pipeline

In this workshop you hydrate a Helm umbrella chart for [Renovate](https://github.com/renovatebot/renovate)
into plain Kubernetes manifests, review chart updates as manifest diffs in pull requests,
promote them from environment to environment and deploy them with Argo CD on a local KIND cluster.

| Lab | Topic | Time |
|-----|-------|------|
| 0 | Set up your fork | 10 min |
| 1 | Hydrate with a prebuilt GitHub workflow | 20 min |
| 2 | Deploy dev with Argo CD on KIND | 25 min |
| 3 | A "harmless" chart update | 20 min |
| 4 | Promotion and environment-specific changes | 15 min |
| 5 | Bonus: look under the hood | open end |

## What you need

- A free GitHub account
- Git and your favourite editor
- Docker

Everything else (`kind`, `kubectl`, `helm`, `yq`) comes with the Steadforce k8s workbench,
a container image started by `./workbench`. It is about 2.5 GB, so **pull it before the workshop**:

```bash
docker pull ghcr.io/steadforce/steadops/workbenches/k8s:sha-9d99d07
```

`./workbench` is a shell script for Linux and macOS. On Windows, run it inside WSL 2.

## The repository

| File | Purpose |
|------|---------|
| `Chart.yaml` | Umbrella chart. Its only dependency is the upstream `renovate` chart. |
| `Chart.lock` | Pins the dependency version. The hydration workflow installs exactly this version. |
| `values.yaml` | Values for all environments |
| `values-dev.yaml`, `values-prod.yaml` | Environment-specific overrides |
| `helm-config.yaml` | Tells the hydration workflow which environments exist, the release name, the namespace and the values files per environment |
| `renovate.json` | Renovate configuration, which would update `Chart.yaml` automatically |
| `argocd/application-dev.yaml` | Argo CD Application for Lab 2 |

---

## Lab 0: Set up your fork (10 min)

1. Fork <https://github.com/steadforce/workshop-helm-hydration-renovate> into your own GitHub account.
2. In your fork, open **Settings → Actions → General** and set:
   - **Actions permissions:** Allow all actions and reusable workflows
   - **Workflow permissions:** Read and write permissions
   - Check **Allow GitHub Actions to create and approve pull requests**
   - Click **Save**
3. Open the **Actions** tab. If GitHub asks you to enable workflows for this fork, enable them.
4. Clone your fork:

   ```bash
   git clone https://github.com/<your-github-user>/workshop-helm-hydration-renovate.git
   cd workshop-helm-hydration-renovate
   ```

5. If git does not know who you are yet, set your identity. Lab 1 creates a commit:

   ```bash
   git config user.name "<your name>"
   git config user.email "<your email>"
   ```

6. If you have not pulled the workbench image yet, start the download now. It runs while you work on Lab 1:

   ```bash
   ./workbench exit
   ```

**Result:** Your fork has only the `main` branch, and Actions are allowed to open pull requests.

---

## Lab 1: Hydrate with a prebuilt GitHub workflow (20 min)

The hydration logic lives in a shared, reusable workflow:
[`steadforce/steadops-workflows`](https://github.com/steadforce/steadops-workflows/blob/v4.5.2/.github/workflows/helm-hydration.yaml).
Your repository only needs a small workflow that calls it.

1. Read `helm-config.yaml`. Which environments will be rendered, and with which values files?
2. Create the workflow directory:

   ```bash
   mkdir -p .github/workflows
   ```

   Then create `.github/workflows/helm-hydration.yaml`:

   ```yaml
   name: Helm hydration

   on:
     push:
       branches:
       - main
     workflow_dispatch:

   permissions:
     contents: write       # create the environment branches and push the rendered manifests
     pull-requests: write  # open one hydration pull request per environment
     issues: write         # create the environment labels

   jobs:
     hydration:
       uses: steadforce/steadops-workflows/.github/workflows/helm-hydration.yaml@v4.5.2
       with:
         helm-version: v4.1.4
   ```

3. Commit and push it:

   ```bash
   git add .github/workflows/helm-hydration.yaml
   git commit -m "Add Helm hydration workflow"
   git push
   ```

4. Watch the run in the **Actions** tab.

**Result:**

- Three new branches: `environments/default`, `environments/dev` and `environments/prod`
- Three pull requests titled `💧 [<environment>] …`, one per environment

**Explore:**

- Open the dev pull request and look at **Files changed**. Everything below `hydrated-manifests/`
  is exactly what will be applied to the cluster: a CronJob, ConfigMaps and a Redis StatefulSet.
- Compare the dev and prod pull requests. Where do they differ, and which values file causes that?

5. Merge all three pull requests. The `environments/*` branches now hold the hydrated state
   of each environment.

---

## Lab 2: Deploy dev with Argo CD on KIND (25 min)

Argo CD watches the `environments/dev` branch. It never runs Helm: it applies the
plain YAML that you reviewed in the pull request.

All commands in this lab run inside the workbench. Start it from the root of your clone:

```bash
./workbench
```

Your clone is mounted at `/workspace`.

1. Create a local cluster and connect the workbench to it:

   ```bash
   kind create cluster --name hydration-workshop
   docker network connect kind "$HOSTNAME"
   kind export kubeconfig --internal --name hydration-workshop
   ```

   When you start `./workbench` again later, it does the last two steps for you.

2. Install Argo CD:

   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd --server-side --force-conflicts \
     -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml
   kubectl -n argocd wait --for=condition=Available deployment --all --timeout=300s
   ```

3. In `argocd/application-dev.yaml`, replace `<your-github-user>` with your GitHub account, then apply it:

   ```bash
   kubectl apply -f argocd/application-dev.yaml
   ```

4. Open the Argo CD UI:

   ```bash
   # Password for the user "admin"
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
   kubectl -n argocd port-forward --address 0.0.0.0 svc/argocd-server 8080:443
   ```

   Open <https://127.0.0.1:8080> and accept the self-signed certificate.
   Use `127.0.0.1`, not `localhost`. Keep the port-forward running and open a second
   terminal with `./workbench` for the next commands.

**Result:** The application `renovate-dev` is **Synced** and **Healthy** after a minute or two.

```bash
kubectl -n renovate get statefulset,cronjob,pods
```

shows the Redis StatefulSet with a running pod and the Renovate CronJob.

---

## Lab 3: A "harmless" chart update (20 min)

Renovate would open a pull request that changes a single line in `Chart.yaml`.
We simulate that by hand.

1. In `Chart.yaml`, change the version from `43.54.0` to `44.1.0`:

   ```yaml
   version: &version 44.1.0
   ```

   The YAML anchor `&version` sets the chart version and the dependency version at once.

   Renovate would also update `Chart.lock`. Do the same inside the workbench:

   ```bash
   helm dependency update
   ```

2. Commit and push to `main`, as Renovate's automerge would. Commit only `Chart.yaml` and
   `Chart.lock`: your `argocd/application-dev.yaml` from Lab 2 stays a local change.

   ```bash
   git commit Chart.yaml Chart.lock -m "Update Helm release renovate to v44.1.0"
   git push
   ```

**Result:** New hydration pull requests titled `💧 [<environment>] renovate 43.54.0 → 44.1.0`.

**Explore the dev pull request:**

- What happened to the files below `hydrated-manifests/renovate/renovate/charts/renovate/charts/redis/`?
- What happened to `RENOVATE_REDIS_URL` in the CronJob?
- `values.yaml` still says `redis.enabled: true`. Why did nobody complain?
- Would you have noticed any of this when reviewing Renovate's one-line change to `Chart.yaml`?

3. **Patch bundling:** Before you merge any of these pull requests, change the version to `44.1.1`, run `helm dependency update` again, then commit `Chart.yaml` and `Chart.lock` and push again.
   Instead of a second set of pull requests, the existing ones are updated: the branch is named
   `hydration-pull-request/<environment>-44.1.x`, so all patch releases of `44.1` land in one pull request.

---

## Lab 4: Promotion and environment-specific changes (15 min)

Each environment branch is promoted on its own, by merging its pull request.

1. **Promote to dev:** Merge only the dev pull request from Lab 3. In Argo CD, click **Refresh**
   or wait up to three minutes.

   **Result:** Argo CD prunes the Redis StatefulSet, Services and ConfigMaps from the cluster.

   ```bash
   kubectl -n renovate get all
   ```

2. **Promote to prod:** When dev looks good, merge the prod pull request (and the default one).

3. **Environment-specific change:** Change a resource limit in `values-prod.yaml`, then commit and push.

   **Result:** Only a prod pull request is opened, because the dev and default manifests did not change.

4. **New environment:** Add a `staging` environment to `helm-config.yaml`:

   ```yaml
   environments:
     # ...
     staging:
       valueFiles:
       - values-prod.yaml
   ```

   **Result:** The workflow creates the branch `environments/staging` and opens a pull request for it.
   You did not touch the workflow.

---

## Lab 5 (bonus): Look under the hood

- [`steadforce/workshop-helm-hydration-workflow`](https://github.com/steadforce/workshop-helm-hydration-workflow)
  builds a hydration workflow step by step. Each step has its own directory in `.github/workflows/`:

  | Step | Building block |
  |------|----------------|
  | 1 | Build a job matrix from the environments in `helm-config.yaml` |
  | 2 | Install Helm and resolve chart dependencies |
  | 3 | Render manifests per environment with `helm template` |
  | 4 | Commit the manifests and open a pull request against the environment branch |
  | 5 | Version-aware branch names |
  | 6 | Bundle patch updates in one pull request |
  | 7 | Argo CD server-side apply and sync waves for CRDs |

- Copy one of the steps into your fork and call it instead of the prebuilt workflow.
- Install the free [Renovate GitHub App](https://github.com/apps/renovate) on your fork and let
  Renovate open the chart update pull request for you. `renovate.json` is already in place.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| The workflow does not start | **Actions** tab: enable workflows for your fork |
| The run fails with `Resource not accessible by integration` | Lab 0, step 2: set **Workflow permissions** to read and write, save, and re-run the workflow |
| The run fails at once on the `uses:` line of the hydration workflow | Actions must allow reusable workflows (Lab 0, step 2), and the version in `uses:` must exist |
| `git commit` says `Author identity unknown` | Lab 0, step 5: set `user.name` and `user.email` |
| `.github/workflows/helm-hydration.yaml` is not picked up | The file must be in `.github/workflows/` at the repository root: `mkdir -p .github/workflows` |
| `GitHub Actions is not permitted to create or approve pull requests` | Lab 0, step 2: check the box and save |
| The hydrate job fails with `the lock file (Chart.lock) is out of sync with the dependencies file (Chart.yaml)` | You changed the version in `Chart.yaml` only. Run `helm dependency update` in the workbench and commit `Chart.lock` too. |
| Argo CD shows `namespaces "renovate" not found` | `destination.namespace` in the Application must be `renovate` |
| Argo CD shows `Repository not found` | Check `repoURL`. Your fork must be public. |
| Redis pod in `ImagePullBackOff` | `values.yaml` must keep the `bitnamilegacy/redis` image override |
| `kubectl` in the workbench: `connection refused` on `127.0.0.1` | Run `docker network connect kind "$HOSTNAME"` and `kind export kubeconfig --internal --name hydration-workshop` |
| Argo CD UI does not load | Use <https://127.0.0.1:8080>, not `localhost`, and keep the port-forward running |
| Argo CD UI does not load although the port-forward runs | Port 8080 is published by the first `./workbench` only. Run the port-forward there. |

## Clean up

```bash
./workbench kind delete cluster --name hydration-workshop
```
