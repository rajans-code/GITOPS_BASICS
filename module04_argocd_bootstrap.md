# Module 04 — Argo CD Bootstrap: App of Apps Pattern

> **Persona:** Dana (DevOps Engineer)  
> **Duration:** ~60 minutes  
> **Prereqs:** Module 03 complete — GitOps repo structured, cluster running

---

## Connecting the Dots

Argo CD is the **reconciliation engine**. It watches a Git repo and ensures the cluster always matches. But here is a chicken-and-egg problem: who deploys Argo CD itself? And once it is running, how does it know about all the other apps?

**Answer: Two-phase bootstrap**

1. **Phase 1 (one-time, manual):** Install Argo CD via Helm. This is the only `helm install` you will ever run manually.
2. **Phase 2 (GitOps forever):** Create a single root ArgoCD Application that points to `apps/root-app.yaml` in your repo. That root app discovers all other Application manifests in `apps/platform/` and `apps/workloads/`. From this point on, adding a new application to the cluster means adding a YAML file to Git — no `helm install`, no `kubectl apply`.

```
Manual Bootstrap
      │
      ▼
 ArgoCD installed
      │
      ▼
 Root Application (apps/root-app.yaml)
      │
      ├── apps/platform/external-secrets.yaml  → ESO installed
      ├── apps/platform/istio.yaml             → Istio installed
      ├── apps/platform/argo-rollouts.yaml     → Rollouts installed
      ├── apps/workloads/backend.yaml          → Backend deployed
      └── apps/workloads/frontend.yaml         → Frontend deployed
```

This is the **App of Apps** pattern. The cluster's entire desired state is expressed in Git, including Argo CD's own configuration.

### Deep Mental Model: Root App vs Child Apps

The phrase **App of Apps** can be confusing at first because every object is an `Application`. The clean way to remember it:

| Layer | What it is | What it points to | What it creates |
|-------|------------|-------------------|-----------------|
| Manual bootstrap | One `kubectl apply` | `apps/root-app.yaml` | The root `Application` CR |
| Root app | An Argo CD `Application` | The `apps/` folder in Git | Child `Application` CRs |
| Child apps | Argo CD `Application` objects | Helm charts, Kustomize folders, or manifest folders | Real Kubernetes resources |

The root app does **not** directly install External Secrets, Rollouts, backend, or frontend. It installs the `Application` objects that describe those things. Then Argo CD reconciles each child `Application` independently.

Think of the root app as a **table of contents**:

```
root-app
  reads apps/
    apps/platform/external-secrets.yaml  -> creates Application/external-secrets
    apps/platform/argo-rollouts.yaml     -> creates Application/argo-rollouts
    apps/workloads/backend.yaml          -> creates Application/backend
    apps/workloads/frontend.yaml         -> creates Application/frontend

Application/external-secrets -> installs ESO Helm chart
Application/argo-rollouts    -> installs Argo Rollouts Helm chart
Application/backend          -> installs local backend Helm chart
Application/frontend         -> installs local frontend Helm chart
```

The important distinction is **who owns what**:

| Object | Owner after bootstrap |
|--------|----------------------|
| `argocd` Helm release | Manual bootstrap, until you later self-manage Argo CD |
| `AppProject/gitops-dev` | Git, but initially applied manually |
| `Application/root-app` | Git, initially applied manually |
| Child `Application` objects | Root app |
| Platform/workload Kubernetes resources | Their child apps |

### The One Sentence to Memorize

> After bootstrap, humans do not install apps into the cluster; humans merge Application manifests into Git, and Argo CD installs the apps.

That is the operational shift. The cluster stops being the place where you make changes. The cluster becomes the place where Argo CD proves Git is true.

### Why Recursive Discovery Matters

The root app points at `path: apps`. By default, Argo CD reads manifests in that directory level. If your child app files live under nested folders such as `apps/platform/` and `apps/workloads/`, the root app must enable recursive directory scanning:

```yaml
spec:
  source:
    repoURL: https://dev.azure.com/AKSAR/GitOpsProject/_git/gitops-config
    targetRevision: main
    path: apps
    directory:
      recurse: true
```

Without `directory.recurse: true`, the root app can appear `Synced / Healthy` while silently ignoring nested child Application files. This is a classic App-of-Apps troubleshooting scenario:

| Symptom | Likely cause |
|---------|--------------|
| `root-app` is Healthy but no child apps exist | Missing `directory.recurse: true` |
| Child apps exist but are OutOfSync | Child app source, chart, or permissions issue |
| Child app is Synced but Degraded | Kubernetes workload/runtime issue |
| Platform app is Healthy but workload app is Degraded | Dependency exists, but workload-specific prerequisites are missing |

### Reconciliation Loop: What Argo CD Actually Does

For each `Application`, Argo CD repeatedly performs this loop:

1. Fetch the source from Git or a Helm repo.
2. Render the desired manifests.
3. Read live resources from the cluster.
4. Compare desired state vs live state.
5. Mark the app `Synced` or `OutOfSync`.
6. If automated sync is enabled, apply the difference.
7. If prune is enabled, delete live resources that were removed from Git.
8. If self-heal is enabled, revert manual cluster drift.

The App-of-Apps pattern simply creates **more Applications** for this loop to manage. The root app has a small desired state: child Application YAMLs. Each child app has a larger desired state: Deployments, Services, CRDs, webhooks, namespaces, and so on.

### Sync Status vs Health Status

Argo CD has two separate opinions about each app:

| Status | Question it answers | Example |
|--------|---------------------|---------|
| Sync | Does live cluster config match Git? | `Synced`, `OutOfSync` |
| Health | Is the thing actually working? | `Healthy`, `Progressing`, `Degraded`, `Missing` |

This distinction matters in interviews and in real operations.

An app can be:

| Combination | Meaning |
|-------------|---------|
| `Synced / Healthy` | Desired state matches Git and resources are working |
| `Synced / Progressing` | Config applied, but rollout is still happening |
| `Synced / Degraded` | Config applied, but the workload is broken |
| `OutOfSync / Healthy` | Workload works, but live config differs from Git |
| `OutOfSync / Missing` | Desired resources are not present in the cluster |

Example: if `backend` points to `backend:PLACEHOLDER` and that image does not exist in ACR, Argo CD can still apply the Deployment. The app may be `Synced` because the manifest matches Git, but `Degraded` because the pod is in `ImagePullBackOff`.

### The Commit Is the Deployment Request

In a GitOps workflow, a merge to `main` is equivalent to saying:

> Please make the cluster match this commit.

That means your pull request is not just code review. It is deployment review. Reviewers should ask:

- What new namespaces will this create?
- What cluster-scoped resources will this create?
- Which Helm repo or Git repo is trusted?
- Which sync wave does this use?
- Could prune delete something important?
- Does this app depend on CRDs that are installed by another app?
- Does this change use placeholders that require a pipeline to replace?

### App-of-Apps Design Rules

Use these rules to stay out of trouble:

| Rule | Why |
|------|-----|
| Keep the root app boring | It should only create child Applications, not business workloads |
| Put platform apps under `apps/platform` | Platform tools usually own CRDs, webhooks, and cluster-wide permissions |
| Put product apps under `apps/workloads` | Workload apps should be easier to delegate to teams |
| Use sync waves for dependencies | CRDs must exist before custom resources |
| Prefer one child app per deployable unit | Easier sync, rollback, ownership, and troubleshooting |
| Pin chart versions | GitOps needs reproducibility |
| Avoid manual `kubectl apply` after bootstrap | It creates drift and weakens Git as source of truth |
| Keep generated secrets out of Git | Use External Secrets or sealed/encrypted secret workflows |

### What "GitOps Forever" Really Means

It does **not** mean nobody ever uses `kubectl`. You still use `kubectl` for inspection and emergency debugging. It means routine desired-state changes are made through Git.

| Action | Good GitOps behavior |
|--------|----------------------|
| Add a new app | Add `apps/workloads/new-app.yaml` |
| Upgrade a Helm chart | Change `targetRevision` in the child Application |
| Change backend replicas | Change the Helm values file in Git |
| Delete an app | Delete its child Application YAML from Git |
| Debug a broken pod | Use `kubectl describe`, `kubectl logs`, Argo CD UI |
| Hotfix a live Deployment with `kubectl edit` | Avoid; Argo CD will self-heal it back |

The rule is: **inspect with kubectl, declare with Git**.

### Configuration Traceback: Read Any Argo CD App Like a Map

To master Argo CD, you must be able to look at an `Application` and answer four questions quickly:

1. **Where is the desired state stored?**
2. **Which exact version of that desired state is Argo CD reading?**
3. **Where will Argo CD apply it in the cluster?**
4. **Which policy controls what Argo CD is allowed to do?**

Every Argo CD `Application` answers those questions in the same fields:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend
  namespace: argocd
spec:
  project: gitops-dev
  source:
    repoURL: https://dev.azure.com/AKSAR/GitOpsProject/_git/gitops-config
    targetRevision: main
    path: charts/backend
    helm:
      releaseName: backend
      valueFiles:
        - ../../clusters/dev/workloads/backend/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: backend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Use this table as your traceback map:

| Field | Meaning | Traceback question |
|-------|---------|-------------------|
| `metadata.name` | Argo CD app name | What app am I looking at in Argo CD? |
| `metadata.namespace` | Namespace where the `Application` CR lives, usually `argocd` | Which Argo CD instance owns this app? |
| `spec.project` | AppProject boundary | What repos, clusters, namespaces, and resources is this app allowed to use? |
| `spec.source.repoURL` | Git repo or Helm repo | Where does desired state come from? |
| `spec.source.targetRevision` | Git branch/tag/commit or Helm chart version | Which version is being reconciled? |
| `spec.source.path` | Folder inside the Git repo | Which folder is rendered? |
| `spec.source.chart` | Helm chart name when using a Helm repo | Which chart is installed? |
| `spec.source.helm.releaseName` | Helm release name Argo CD uses while rendering | What names will chart templates often derive from? |
| `spec.source.helm.valueFiles` | Values overlays used with a Git-hosted chart | Which environment-specific config is merged? |
| `spec.source.helm.values` | Inline Helm values | What values are embedded directly in the app manifest? |
| `spec.destination.server` | Kubernetes API target | Which cluster receives the resources? |
| `spec.destination.namespace` | Default namespace for namespaced resources | Where do namespaced resources land? |
| `spec.syncPolicy.automated` | Enables automatic sync | Will Argo CD apply changes without a human clicking sync? |
| `spec.syncPolicy.automated.prune` | Deletes resources removed from Git | Will removed Git objects be deleted from the cluster? |
| `spec.syncPolicy.automated.selfHeal` | Reverts manual cluster drift | Will manual `kubectl edit` changes be undone? |
| `spec.syncPolicy.syncOptions` | Special sync behavior | Does Argo CD create namespaces, use server-side apply, etc.? |
| `spec.ignoreDifferences` | Fields Argo CD ignores during diff | Which live changes are intentionally not considered drift? |

#### Bootstrap App Traceback

For `root-app`, read the important fields like this:

```yaml
spec:
  project: gitops-dev
  source:
    repoURL: https://dev.azure.com/AKSAR/GitOpsProject/_git/gitops-config
    targetRevision: main
    path: apps
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
```

Traceback:

| Field | Meaning in root-app |
|-------|---------------------|
| `repoURL` | Read child Application manifests from `gitops-config` |
| `targetRevision: main` | Use the approved mainline desired state |
| `path: apps` | Start scanning in the `apps/` folder |
| `directory.recurse: true` | Include nested app files in `apps/platform` and `apps/workloads` |
| `destination.namespace: argocd` | Create child Application CRs in the `argocd` namespace |
| `project: gitops-dev` | Root app must obey the `gitops-dev` AppProject restrictions |

If children are missing, the first traceback is:

```powershell
kubectl get application root-app -n argocd -o yaml
```

Then verify:

```powershell
git -C C:\MyProjects\GitOps_basics\gitops-repo ls-files apps
```

The root app cannot deploy files that are not committed and pushed to the branch named in `targetRevision`.

#### Child App Traceback: Git-Hosted Helm Chart

Backend and frontend use charts stored inside the same GitOps repo:

```yaml
spec:
  source:
    repoURL: https://dev.azure.com/AKSAR/GitOpsProject/_git/gitops-config
    targetRevision: main
    path: charts/backend
    helm:
      releaseName: backend
      valueFiles:
        - ../../clusters/dev/workloads/backend/values.yaml
```

Traceback:

1. Clone/read `repoURL`.
2. Checkout `targetRevision`.
3. Enter `path`, here `charts/backend`.
4. Render the Helm chart.
5. Merge `valueFiles`, resolved relative to `charts/backend`.
6. Apply rendered resources to `destination.namespace`.

Debug commands:

```powershell
argocd app get backend --core
argocd app manifests backend --core
helm template backend .\charts\backend -f .\clusters\dev\workloads\backend\values.yaml
kubectl get all -n backend
```

If `argocd app manifests backend --core` fails, the problem is usually repo access, path, Helm rendering, values file path, or AppProject restrictions. If manifests render but pods fail, the problem is Kubernetes runtime: image pull, secret, probe, resources, RBAC, or scheduling.

#### Child App Traceback: Remote Helm Repo

External Secrets and Argo Rollouts use remote Helm repos:

```yaml
spec:
  source:
    repoURL: https://charts.external-secrets.io
    chart: external-secrets
    targetRevision: 0.9.20
    helm:
      releaseName: external-secrets
      values: |
        installCRDs: true
```

Traceback:

| Field | Meaning |
|-------|---------|
| `repoURL` | Helm chart repository |
| `chart` | Chart name inside that repository |
| `targetRevision` | Chart version, not Git branch |
| `helm.values` | Inline chart values |
| `destination.namespace` | Namespace where chart resources are installed |

This is a common interview nuance: `targetRevision` means different things depending on source type.

| Source type | `targetRevision` means |
|-------------|------------------------|
| Git repo | Branch, tag, or commit SHA |
| Helm repo | Chart version |
| OCI Helm repo | Chart version/tag |

#### AppProject Traceback

When an app is denied or cannot sync, inspect the project:

```powershell
kubectl get appproject gitops-dev -n argocd -o yaml
```

Trace these fields:

| AppProject field | What to verify |
|------------------|----------------|
| `sourceRepos` | Does it allow the app's `spec.source.repoURL`? |
| `destinations.server` | Does it allow the target cluster? |
| `destinations.namespace` | Does it allow the target namespace? |
| `clusterResourceWhitelist` | Can the app create CRDs, ClusterRoles, webhooks? |
| `namespaceResourceWhitelist` | Can the app create namespaced resources? |
| `roles` | Who can get, sync, or administer apps in this project? |

If source repo, destination, or resource kind is not allowed by the project, Argo CD will block the app even if the YAML is otherwise correct.

#### Repo Credential Traceback

For private repos, Argo CD needs credentials. In this lab, the repo credential is stored as a Secret:

```powershell
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=repository
argocd repo list --core
```

Trace these fields:

| Secret data | Purpose |
|-------------|---------|
| `url` | Must match `spec.source.repoURL` |
| `username` | Azure DevOps username or service identity label |
| `password` | PAT or token |
| `type` | Usually `git` |
| `name` | Friendly repo name in Argo CD |

If `repoURL` differs even slightly, such as trailing slash or different project path, Argo CD may not match the credential you expect.

#### Destination Traceback

For this lab:

```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: backend
```

This means "deploy into the same cluster where Argo CD is running." For multi-cluster GitOps, `server` may point to another cluster registered in Argo CD.

Useful commands:

```powershell
argocd cluster list --core
kubectl config current-context
kubectl get ns backend
```

If the namespace does not exist, check whether this sync option is present:

```yaml
syncOptions:
  - CreateNamespace=true
```

#### Target Revision Traceback

Always ask: "What exact desired-state version is this app reading?"

```powershell
kubectl get application backend -n argocd -o jsonpath="{.spec.source.targetRevision}"
kubectl get application backend -n argocd -o jsonpath="{.status.sync.revision}"
```

Difference:

| Field | Meaning |
|-------|---------|
| `spec.source.targetRevision` | Desired target, such as `main` or `feature/kiali-upgrade` |
| `status.sync.revision` | Actual resolved revision Argo CD last compared/synced |

For a Git branch, `targetRevision` may be `main`, but `status.sync.revision` should be a commit SHA. That SHA tells you exactly what commit is running.

#### Full Traceback Drill

When someone asks, "Why is this running in the cluster?", perform this drill:

```powershell
# 1. Identify the app and source
argocd app get backend --core

# 2. Find exact Git revision
kubectl get application backend -n argocd -o jsonpath="{.status.sync.revision}"

# 3. Confirm repo/path/target
kubectl get application backend -n argocd -o yaml

# 4. Render what Argo CD should apply
argocd app manifests backend --core

# 5. Compare live resources
kubectl get all -n backend
kubectl describe deployment backend-backend -n backend

# 6. Check recent failures
kubectl get events -n backend --sort-by=.lastTimestamp
```

Your mental flow should be:

```text
Application -> Project -> Repo credential -> Source repo -> Target revision -> Path/chart -> Values -> Rendered manifests -> Destination cluster/namespace -> Live resources -> Events/logs
```

That chain is the core operational skill for Argo CD.

---

## 1. Install Argo CD (Bootstrap — One Time)

```powershell
. C:\MyProjects\GitOps_basics\vars.ps1
kubectl config use-context gitops-dev

# Create namespace
kubectl create namespace argocd

# Add Argo CD Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD with production-appropriate settings
helm install argocd argo/argo-cd `
  --namespace argocd `
  --version 7.3.4 `
  --set server.replicas=2 `
  --set applicationSet.replicas=2 `
  --set configs.params."server.insecure"=false `
  --set configs.params."server.disable.auth"=false `
  --wait --timeout 10m
```

> **Version pinning:** Always specify `--version` for Argo CD in production. Floating to `latest` means a Helm repo update can silently upgrade your GitOps engine.

### 1.1 Get the Initial Admin Password

```powershell
# The initial password is auto-generated and stored in a secret
$ARGOCD_PWD = kubectl get secret -n argocd argocd-initial-admin-secret `
  -o jsonpath="{.data.password}" | `
  ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

Write-Host "ArgoCD initial password: $ARGOCD_PWD"
```

### 1.2 Access the ArgoCD UI

```powershell
# Port-forward (local access only — secure for lab)
kubectl port-forward svc/argocd-server -n argocd 8443:443
```

Open browser: https://localhost:8443  
Login: `admin` / `<password from above>`

### 1.3 Change the Admin Password

```powershell
argocd login localhost:8443 --username admin --password $ARGOCD_PWD --insecure

# Change to something you'll remember
argocd account update-password `
  --current-password $ARGOCD_PWD `
  --new-password "GitOps@Secure2025!"
```

> **Enterprise Practice:** In production, admin login is disabled and access is via SSO (Azure AD OIDC). We cover that in Day-2 Module 10. For now, change the auto-generated password immediately.

---

## 2. Connect ArgoCD to the ADO GitOps Repo

ArgoCD needs a credential to pull from your private ADO Git repo.

### 2.1 Retrieve the ADO PAT from Key Vault

The read-only ArgoCD PAT (`ado-argocd-pat`) was stored in Key Vault in Module 01, section 5.2. Retrieve it now:

```powershell
. C:\MyProjects\GitOps_basics\vars.ps1

$ADO_PAT = az keyvault secret show --vault-name $KV --name "ado-argocd-pat" --query "value" -o tsv
```

### 2.2 Register the Repo with ArgoCD

```powershell
argocd repo add $GITOPS_REPO_URL `
  --username "argocd" `
  --password "$ADO_PAT" `
  --name "gitops-config"
```

Verify:
```powershell
argocd repo list
```

---

## 3. Create the ArgoCD Project

An ArgoCD **Project** is a scope boundary. It controls which repos, clusters, and namespaces an Application can access. This is key for multi-team enterprise environments.

### 3.1 Define the AppProject YAML

```powershell
Set-Location C:\MyProjects\GitOps_basics\gitops-repo\clusters\dev\argocd

@"
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: gitops-dev
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: "GitOps Dev Environment — AKSAR Project"

  # Source repos this project is allowed to sync from
  sourceRepos:
    - "https://dev.azure.com/AKSAR/GitOpsProject/_git/gitops-config"
    - "https://argoproj.github.io/argo-helm"
    - "https://charts.external-secrets.io"
    - "https://istio-release.storage.googleapis.com/charts"
    - "https://argoproj.github.io/argo-helm"
    - "https://helm.nginx.com/stable"

  # Allowed destination clusters and namespaces
  destinations:
    - server: https://kubernetes.default.svc   # in-cluster
      namespace: "*"

  # Allow creating cluster-scoped resources (CRDs etc.)
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"

  # Namespace-scoped resource restrictions (allow everything for now)
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"

  # RBAC roles within this project
  roles:
    - name: developer
      description: Developer — can sync workloads, cannot manage platform
      policies:
        - p, proj:gitops-dev:developer, applications, get, gitops-dev/workloads/*, allow
        - p, proj:gitops-dev:developer, applications, sync, gitops-dev/workloads/*, allow
      groups:
        - developers   # maps to an Azure AD group via SSO (Day-2)

    - name: devops
      description: DevOps — full access to all apps in project
      policies:
        - p, proj:gitops-dev:devops, applications, *, gitops-dev/*, allow
      groups:
        - devops-engineers

    - name: sre
      description: SRE — read-only by default, can hard-refresh
      policies:
        - p, proj:gitops-dev:sre, applications, get, gitops-dev/*, allow
        - p, proj:gitops-dev:sre, applications, action/*, gitops-dev/*, allow
      groups:
        - sre-team
"@ | Out-File -FilePath appproject.yaml -Encoding UTF8
```

**Replace `$GITOPS_REPO_URL` placeholder:**
```powershell
. C:\MyProjects\GitOps_basics\vars.ps1
(Get-Content appproject.yaml) -replace '\$GITOPS_REPO_URL', $GITOPS_REPO_URL | Set-Content appproject.yaml
```

---

## 4. Create the App of Apps Root Application

### 4.1 root-app.yaml

```powershell
Set-Location C:\MyProjects\GitOps_basics\gitops-repo\apps

@"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # sync before children
spec:
  project: gitops-dev
  source:
    repoURL: "$GITOPS_REPO_URL"
    targetRevision: main
    path: apps
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true      # remove resources deleted from Git
      selfHeal: true   # revert manual kubectl changes
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
"@ | Out-File -FilePath root-app.yaml -Encoding UTF8

(Get-Content root-app.yaml) -replace '\$GITOPS_REPO_URL', $GITOPS_REPO_URL | Set-Content root-app.yaml
```

> **Key Settings Explained:**
> - `prune: true` — if you delete a manifest from Git, Argo CD removes it from the cluster. This is how GitOps provides a clean cluster.
> - `selfHeal: true` — if someone runs `kubectl edit` directly, Argo CD reverts it within 3 minutes. This enforces the "Git is truth" rule.
> - `sync-wave: "-1"` — ensures the root app syncs before all its children.

### 4.2 Platform App Definitions

```powershell
Set-Location C:\MyProjects\GitOps_basics\gitops-repo\apps\platform

# External Secrets Operator
@"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: gitops-dev
  source:
    repoURL: https://charts.external-secrets.io
    chart: external-secrets
    targetRevision: 0.9.20
    helm:
      releaseName: external-secrets
      values: |
        installCRDs: true
        webhook:
          create: true
        certController:
          create: true
        serviceAccount:
          create: true
          name: external-secrets-sa
          annotations:
            azure.workload.identity/client-id: "REPLACE_UAMI_CLIENT_ID"
        podLabels:
          azure.workload.identity/use: "true"
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
"@ | Out-File -FilePath external-secrets.yaml -Encoding UTF8

# Argo Rollouts
@"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-rollouts
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: gitops-dev
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo-rollouts
    targetRevision: 2.37.0
    helm:
      releaseName: argo-rollouts
      values: |
        installCRDs: true
        controller:
          replicas: 2
  destination:
    server: https://kubernetes.default.svc
    namespace: argo-rollouts
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
"@ | Out-File -FilePath argo-rollouts.yaml -Encoding UTF8
```

**Inject UAMI client ID:**
```powershell
. C:\MyProjects\GitOps_basics\vars.ps1
$UAMI_CLIENT = az identity show --resource-group $RG --name $UAMI --query "clientId" -o tsv

(Get-Content external-secrets.yaml) -replace "REPLACE_UAMI_CLIENT_ID", $UAMI_CLIENT | Set-Content external-secrets.yaml
```

### 4.3 Workload App Definitions

```powershell
Set-Location C:\MyProjects\GitOps_basics\gitops-repo\apps\workloads

# Backend Application
@"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"   # sync after platform components
spec:
  project: gitops-dev
  source:
    repoURL: "$GITOPS_REPO_URL"
    targetRevision: main
    path: charts/backend
    helm:
      releaseName: backend
      valueFiles:
        - ../../clusters/dev/workloads/backend/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: backend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas   # HPA manages replicas — ignore drift on this field
"@ | Out-File -FilePath backend.yaml -Encoding UTF8

# Frontend Application
@"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: gitops-dev
  source:
    repoURL: "https://dev.azure.com/AKSAR/GitOpsProject/_git/gitops-config"
    targetRevision: main
    path: charts/frontend
    helm:
      releaseName: frontend
      valueFiles:
        - ../../clusters/dev/workloads/frontend/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: frontend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
"@ | Out-File -FilePath frontend.yaml -Encoding UTF8

# Replace GITOPS_REPO_URL placeholder in all workload app definitions
. C:\MyProjects\GitOps_basics\vars.ps1
Get-ChildItem -Filter "*.yaml" | ForEach-Object {
    (Get-Content $_.FullName) -replace '\$GITOPS_REPO_URL', $GITOPS_REPO_URL | Set-Content $_.FullName
}
```

> **`ignoreDifferences` for replicas:** HPA continuously adjusts `spec.replicas`. Without this setting, ArgoCD would see a "drift" every time HPA changes the replica count and would reset it back to the value in Git — fighting with HPA. The `ignoreDifferences` rule tells ArgoCD: "this specific field is managed by the cluster, not by Git."

---

## 5. Bootstrap the Root Application

This is the last manual step. After this, all changes are via Git PRs.

```powershell
Set-Location C:\MyProjects\GitOps_basics\gitops-repo

# Commit everything first
git add .
git commit -m "feat: argocd app-of-apps structure, project rbac, platform and workload apps"
git push origin main

# Apply the AppProject and root Application manually (bootstrap)
kubectl apply -f clusters/dev/argocd/appproject.yaml
kubectl apply -f apps/root-app.yaml
```

Watch ArgoCD discover and sync all apps:

```powershell
# Watch sync progress
kubectl get applications -n argocd -w
```

Or in the ArgoCD UI: https://localhost:8443 — you should see the root app appear, then platform apps, then workloads.

---

## 6. ArgoCD Sync Waves — Order of Operations

The `argocd.argoproj.io/sync-wave` annotation controls the order in which resources are applied during a sync:

| Wave | Resources | Why |
|------|-----------|-----|
| `-1` | root-app | Must exist before children |
| `0` | external-secrets, argo-rollouts | Platform tools needed by apps |
| `1` | ClusterSecretStore | Needs ESO CRDs from wave 0 |
| `2` | backend, frontend | Apps need secrets (wave 1) ready |

```powershell
# Verify sync wave order from GitOps repo
Get-ChildItem .\apps -Recurse -Filter *.yaml | Select-String -Pattern "sync-wave" | Format-Table
```

---

## 7. Mastery Notes: Nuances That Matter

### Sync Waves Are Ordering Hints, Not Health Gates

Sync waves control the order Argo CD **applies** resources during a sync. They do not always mean Argo CD waits for every dependency to become fully ready before moving on.

For example:

| Wave | Example | Nuance |
|------|---------|--------|
| `0` | External Secrets Operator | CRDs may be created before the webhook is fully ready |
| `1` | `ClusterSecretStore` | The CRD must exist first, but provider auth may still fail |
| `2` | Backend | Deployment can apply even if its image tag does not exist yet |

If a child app is `Synced / Degraded`, sync waves probably did their job. The problem is usually runtime readiness, missing images, missing secrets, bad RBAC, or a failing webhook.

### Root App Healthy Does Not Mean All Children Are Healthy

The root app is responsible for child `Application` manifests. If root-app is `Healthy`, it means Argo CD successfully created or updated those Application objects. It does not guarantee every child app is healthy.

Always validate both layers:

```powershell
argocd app get root-app --core
kubectl get applications -n argocd -o wide
```

Use this reading:

| Thing to inspect | What it tells you |
|------------------|-------------------|
| `root-app` | Did Argo CD discover and create the child Applications? |
| `external-secrets` app | Did the ESO chart and CRDs install? |
| `external-secrets-store` app | Did the `ClusterSecretStore` get created? |
| `backend` app | Did the backend chart render and deploy? |
| backend pod | Did the image pull and container start? |
| backend `ExternalSecret` | Did ESO read from Key Vault and create the Kubernetes Secret? |

### AppProject Is the Safety Boundary

`AppProject` is not just metadata. It is one of the main guardrails in Argo CD.

It controls:

- Which Git or Helm repos apps may pull from.
- Which destination clusters apps may deploy to.
- Which namespaces apps may deploy to.
- Whether cluster-scoped resources are allowed.
- Which users or groups can sync apps in that project.

If a child app says `PermissionDenied`, check the project before checking the chart.

Common project failures:

| Error shape | Likely cause |
|-------------|--------------|
| repo is not permitted | `sourceRepos` does not include the repo URL |
| destination not permitted | `destinations` does not include namespace or cluster |
| cluster-scoped resource denied | `clusterResourceWhitelist` is too restrictive |
| user cannot sync | project role policy/group mapping is missing |

### Prune Is Powerful

`prune: true` means Argo CD can delete resources that disappear from Git. This is correct GitOps behavior, but it should make reviewers pay attention.

Example:

1. `apps/workloads/backend.yaml` exists in Git.
2. Argo CD deploys backend.
3. Someone deletes `apps/workloads/backend.yaml` and merges to `main`.
4. Root app prunes the `Application/backend` object.
5. Because `Application/backend` has a finalizer, Argo CD also deletes backend resources.

This is why Git history becomes the audit trail for both creation and deletion.

### Self-Heal Is a Drift Corrector

`selfHeal: true` means if someone changes a managed resource directly in the cluster, Argo CD will restore the Git version.

Good use:

- Prevents long-lived manual drift.
- Forces real fixes through pull requests.
- Makes environments reproducible.

Danger:

- If Git is wrong, Argo CD will faithfully keep applying the wrong thing.
- Emergency manual patches will be reverted unless you also update Git or pause automation.

For emergency work, a mature team documents one of these:

- Temporarily disable auto-sync for the app.
- Patch live state and immediately raise a PR to match it.
- Revert the bad Git commit.

### Local Helm Value File Paths Are Relative to the Chart

For workload apps in this module:

```yaml
source:
  path: charts/backend
  helm:
    valueFiles:
      - ../../clusters/dev/workloads/backend/values.yaml
```

That relative path is resolved from the chart path, not from the repo root. Since the chart path is `charts/backend`, `../../clusters/...` walks back to the repo root and then into the environment overlay.

If this path is wrong, the child app usually fails with a manifest generation error.

### PLACEHOLDER Images Are a Pipeline Contract

In this learning repo, values files use:

```yaml
image:
  tag: "PLACEHOLDER"
```

That is a deliberate signal: CI/CD should replace the tag after building and pushing an image. Until that happens, Argo CD can apply the Deployment, but Kubernetes pods may show `ImagePullBackOff`.

Do not confuse these:

| Symptom | Layer |
|---------|-------|
| App cannot render manifests | Argo CD/source problem |
| App is Synced but pod has `ImagePullBackOff` | Image registry/pipeline problem |
| App is Degraded because ExternalSecret cannot read Key Vault | Secret provider/auth problem |

### SRE Feature-Branch Testing: Point One App at a Branch

A common enterprise workaround is to let SREs test a newer version of a platform tool from a feature branch without changing the default `main` desired state for everyone.

Scenario:

> The SRE team wants to test a newer Kiali version before promoting it to `main`.

The safe workflow is:

1. Create a Git branch:

```powershell
git checkout -b feature/kiali-upgrade
```

2. Update only the Kiali configuration on that branch. Depending on how the repo is structured, this might be:

```yaml
# apps/platform/kiali.yaml
source:
  repoURL: https://kiali.org/helm-charts
  chart: kiali-server
  targetRevision: 2.0.0
```

or:

```yaml
# clusters/dev/platform/kiali/values.yaml
image:
  tag: v2.0.0
```

or, in Terraform-driven platforms:

```hcl
kiali_version = "2.0.0"
```

3. Push the branch:

```powershell
git push origin feature/kiali-upgrade
```

4. Temporarily point the Argo CD app at that branch:

```yaml
spec:
  source:
    repoURL: https://dev.azure.com/AKSAR/GitOpsProject/_git/gitops-config
    targetRevision: feature/kiali-upgrade
    path: apps/platform/kiali
```

5. Let Argo CD sync and validate the test deployment.

6. If the test passes, open a PR from `feature/kiali-upgrade` into `main`.

7. After merge, point `targetRevision` back to `main`, or remove the temporary override if the app normally inherits `main`.

This pattern lets a team test a future desired state while keeping the stable environment anchored to `main`.

#### Same Pattern with an Addons Repo Variable

Some enterprises do not edit Argo CD `targetRevision` directly in every Application. Instead, they parameterize the addons repo branch or tag with a variable such as:

```hcl
target_addons_repo_version = "main"
```

or, in older naming conventions:

```hcl
addons_repo_version = "master"
```

For a Kiali test, SRE can temporarily change:

```hcl
target_addons_repo_version = "feature/kiali-upgrade"
```

Then the generated or templated Argo CD Application points to the feature branch:

```yaml
spec:
  source:
    repoURL: https://dev.azure.com/AKSAR/GitOpsProject/_git/gitops-config
    targetRevision: feature/kiali-upgrade
    path: apps/platform/kiali
```

The concept is the same:

| Stable mode | Test mode |
|-------------|-----------|
| `targetRevision: main` | `targetRevision: feature/kiali-upgrade` |
| `target_addons_repo_version = "main"` | `target_addons_repo_version = "feature/kiali-upgrade"` |
| Everyone gets the approved version | One environment tests the candidate version |

#### Why This Is Useful

- SRE can test a platform upgrade without merging risky changes to `main`.
- Argo CD still performs the deployment, so the test remains GitOps-native.
- The branch name becomes an audit trail for the experiment.
- Rollback is simple: point back to `main` and sync.
- The same pattern works for Kiali, Istio, External Secrets, Argo Rollouts, ingress controllers, observability agents, and policy tools.

#### Critical Nuance: Branch Overrides Are Temporary

A feature branch override is a useful test lever, but it must not become permanent production configuration.

Bad long-term state:

```yaml
targetRevision: feature/kiali-upgrade
```

Why it is risky:

- Feature branches can be deleted.
- Feature branches may not have branch protections.
- Other unreviewed commits may be pushed to the branch.
- The running cluster no longer tracks the approved `main` state.

Good final state after validation:

```yaml
targetRevision: main
```

with the tested Kiali version merged into `main`.

#### Operational Guardrails

Use these rules when testing platform tools from branches:

| Guardrail | Reason |
|-----------|--------|
| Use a clearly named branch, such as `feature/kiali-upgrade` | Makes the intent obvious in Argo CD and Git history |
| Limit the override to a dev or staging environment | Avoids unreviewed production drift |
| Pin the chart/app version inside the branch | Avoids testing a moving `latest` target |
| Require a PR before merging to `main` | Keeps the final desired state reviewed |
| Revert `targetRevision` back to `main` after the test | Restores the normal GitOps contract |
| Record the test result in the PR | Makes the upgrade decision auditable |

#### How to Verify What Branch Argo CD Is Using

```powershell
argocd app get kiali --core
```

Look for:

```text
Repo:   https://dev.azure.com/AKSAR/GitOpsProject/_git/gitops-config
Target: feature/kiali-upgrade
Path:   apps/platform/kiali
```

Or through Kubernetes:

```powershell
kubectl get application kiali -n argocd -o jsonpath="{.spec.source.targetRevision}"
```

#### Interview Answer

**Question:** How can an SRE team test a newer platform addon version without merging to `main`?

**Answer:** Create a feature branch, update the addon version on that branch, and temporarily point the relevant Argo CD Application `targetRevision` or the platform variable such as `target_addons_repo_version` to that feature branch. Argo CD then reconciles from the branch. If validation passes, merge the branch into `main` and point the Application back to `main`. This keeps testing GitOps-native while preserving `main` as the stable source of truth.

### Free Trial Lab Adjustment

The original enterprise-style defaults use multiple replicas. For this Azure free trial lab, we intentionally reduce app replicas to `1` and avoid extra node pools unless needed. That means the self-heal validation in this environment should expect the Git value, which may be `1`, not always `2`.

Check the actual Git value before running a self-heal test:

```powershell
helm template backend .\charts\backend -f .\clusters\dev\workloads\backend\values.yaml |
  Select-String "replicas:"
```

---

## 8. Troubleshooting Playbook

Use this sequence when App-of-Apps is not behaving.

### 8.1 Root App Does Not Create Children

```powershell
kubectl get application root-app -n argocd -o yaml
argocd app get root-app --core
```

Check:

- `spec.source.path` points to `apps`.
- `spec.source.directory.recurse` is `true`.
- The child YAMLs are committed and pushed to the branch in `targetRevision`.
- The repository credential is successful.

Verify files from the repo:

```powershell
Get-ChildItem .\apps -Recurse -Filter *.yaml
```

### 8.2 Child App Exists But Is OutOfSync

```powershell
kubectl describe application backend -n argocd
argocd app diff backend --core
argocd app get backend --core
```

Look for:

- Bad Helm values path.
- Chart version not found.
- Repo not allowed by AppProject.
- Destination namespace not allowed.
- CRD missing.

### 8.3 Child App Is Synced But Degraded

```powershell
kubectl get pods -n backend
kubectl describe pod -n backend -l app.kubernetes.io/name=backend
kubectl get events -n backend --sort-by=.lastTimestamp
```

Likely causes:

- Image tag does not exist.
- ACR pull permission missing.
- Secret missing.
- ExternalSecret provider auth failed.
- Container crashes after start.
- Readiness probe path is wrong.

### 8.4 Repository Is Not Connected

```powershell
argocd repo list --core
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=repository
```

A private Azure DevOps repo needs a repository credential. In this lab, that is represented as an Argo CD repository secret in the `argocd` namespace.

### 8.5 Verify Sync Waves

Run from `C:\MyProjects\GitOps_basics\gitops-repo`:

```powershell
Get-ChildItem .\apps -Recurse -Filter *.yaml |
  Select-String -Pattern "sync-wave" |
  Format-Table Path,LineNumber,Line -AutoSize
```

Expected pattern:

| App | Wave |
|-----|------|
| root-app | `-1` |
| external-secrets | `0` |
| argo-rollouts | `0` |
| external-secrets-store | `1` |
| backend | `2` |
| frontend | `2` |

---

## 9. Interview Q&A: App of Apps

### Q1. What is the App-of-Apps pattern in Argo CD?

It is a bootstrap pattern where one root Argo CD `Application` points to a Git folder containing other `Application` manifests. The root app creates the child apps, and each child app manages a platform component or workload.

### Q2. Why not apply every Application manually with `kubectl apply`?

Manual apply does not scale and weakens Git as the source of truth. With App-of-Apps, adding or removing an app is a Git change. The root app discovers the change and Argo CD reconciles it automatically.

### Q3. What is the only manual step after Argo CD is installed?

Apply the `AppProject` and the root `Application` one time:

```powershell
kubectl apply -f clusters/dev/argocd/appproject.yaml
kubectl apply -f apps/root-app.yaml
```

After that, routine changes should flow through Git.

### Q4. What does `directory.recurse: true` do?

It tells Argo CD to scan nested folders under the root app source path. Without it, a root app pointed at `apps` may not discover files in `apps/platform` or `apps/workloads`.

### Q5. What is the difference between the root app and a child app?

The root app manages Application objects. A child app manages actual platform or workload resources such as Deployments, Services, CRDs, Ingresses, and Helm releases.

### Q6. What problem do sync waves solve?

They provide apply ordering. Platform dependencies like CRDs and controllers should apply before custom resources and workloads that rely on them.

### Q7. Do sync waves guarantee readiness?

Not completely. They order sync tasks, but runtime health can still lag. A CRD may exist while the controller webhook is still starting. A Deployment may apply while its image is unavailable.

### Q8. What is the difference between `Synced` and `Healthy`?

`Synced` means live manifests match Git. `Healthy` means the resources are operational according to Argo CD health checks. A Deployment can be Synced but Degraded if its pod cannot pull an image.

### Q9. What does `prune: true` do?

It deletes live resources that were removed from Git. This keeps the cluster clean and prevents orphaned resources, but it also means deletions in Git are real cluster deletions.

### Q10. What does `selfHeal: true` do?

It tells Argo CD to revert manual changes in the cluster back to the Git-defined state. This enforces Git as the source of truth.

### Q11. Why use an AppProject?

An AppProject scopes what apps can do. It limits allowed source repos, destination namespaces/clusters, cluster-scoped resources, and RBAC permissions. It is a guardrail for multi-team GitOps.

### Q12. How do you add a new workload after bootstrap?

1. Add a chart or manifest folder.
2. Add an environment values file if needed.
3. Add a child `Application` YAML under `apps/workloads`.
4. Commit, push, and merge.
5. Argo CD discovers and syncs it.

No `helm install` and no routine `kubectl apply`.

### Q13. How do you remove an app?

Delete its child `Application` YAML from Git. With prune and finalizers enabled, Argo CD removes the Application and its managed resources.

### Q14. Why can a child app be `Synced / Degraded`?

Because desired manifests were applied successfully, but the runtime failed. Common causes are missing images, bad probes, missing secrets, insufficient resources, or external provider failures.

### Q15. How would you debug an App-of-Apps issue in production?

Start from the top:

```powershell
argocd app get root-app --core
kubectl get applications -n argocd -o wide
kubectl describe application <child-app> -n argocd
kubectl get events -A --sort-by=.lastTimestamp
```

Then inspect the specific namespace and resource that is unhealthy.

### Q16. What should be reviewed in a PR that adds a new child Application?

Review the repo URL, target revision, chart path, values files, destination namespace, sync policy, prune behavior, sync wave, AppProject permissions, and whether the app creates cluster-scoped resources.

### Q17. What is a common beginner mistake with App-of-Apps?

Pointing the root app at `apps` but forgetting recursive discovery. The root app then looks healthy while no nested child apps are created.

### Q18. What is the difference between App-of-Apps and ApplicationSet?

App-of-Apps uses plain Application manifests managed by a root app. ApplicationSet generates Application objects from templates and generators, such as a list of clusters or directories. App-of-Apps is simpler to learn; ApplicationSet is more scalable for many similar apps or many clusters.

### Q19. Should Argo CD manage itself?

Eventually, yes, many teams make Argo CD self-managed. But initial installation is usually a controlled bootstrap step. This module keeps Phase 1 manual and Phase 2 GitOps-managed for clarity.

### Q20. What is the safest mental model for GitOps?

Git is the desired state. Argo CD is the reconciler. Kubernetes is the runtime. Humans propose state changes through pull requests.

---

## 10. Validation Checklist — Module 04

```powershell
# Check 1: All applications Healthy and Synced
kubectl get applications -n argocd -o wide

# Check 2: AppProject created
kubectl get appprojects -n argocd

# Check 3: Root app synced
argocd app get root-app

# Check 4: Platform apps running
kubectl get pods -n external-secrets
kubectl get pods -n argo-rollouts

# Check 5: Workload namespaces created
kubectl get namespaces | Select-String "backend|frontend"

# Check 6: selfHeal test — manually change a replica count, then watch Argo revert it
kubectl scale deployment backend-backend --replicas=5 -n backend
Start-Sleep -Seconds 180
kubectl get deployment backend-backend -n backend -o jsonpath="{.spec.replicas}"
# Expected: back to the replica count defined in Git. In the free-trial lab profile, this is usually 1.
```

| Check | Expected |
|-------|----------|
| All applications | `Synced / Healthy` |
| AppProject `gitops-dev` | Present |
| `external-secrets` pods | 3 pods `Running` |
| `argo-rollouts` pods | 2 pods `Running` |
| selfHeal test | Replicas reset to Git value within 3 min |

---

## Key Concepts Recap

| Concept | Why It Matters |
|---------|---------------|
| App of Apps | One PR to add an app to the cluster — no direct cluster access needed |
| `selfHeal: true` | Enforces "Git is truth" — manual kubectl changes are reverted |
| `prune: true` | Deleted manifests in Git → deleted resources in cluster. No orphan cruft. |
| `ignoreDifferences` for replicas | Prevents ArgoCD fighting HPA — each tool manages its own concern |
| Sync waves | Dependencies respected even when all apps sync simultaneously |
| ArgoCD Projects + RBAC | Developers can only sync their own workloads; platform is SRE-only |

---

*Next → [Module 05 — External Secrets Operator & Azure Key Vault](module05_external_secrets.md)*
