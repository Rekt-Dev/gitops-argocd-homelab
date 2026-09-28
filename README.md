# gitops-argocd-homelab

ArgoCD-based GitOps design for a kubeadm homelab cluster. All cluster application state is declared in this repository — no manual `kubectl apply` beyond the initial bootstrap. Uses the App of Apps pattern to manage multiple application repositories from a single control point.

## Cluster Context

- **Nodes:** k8s (control plane, 192.168.1.110), k8s1, k8s2
- **CNI:** Calico
- **Kubernetes version:** v1.35.x

## Repository Structure

```
install/
  argocd-values.yaml       Helm values for ArgoCD installation
apps/
  app-of-apps.yaml         Root application — manages all other Application CRDs
  production-app.yaml      Deploys k8s-production-patterns workloads
  monitoring-app.yaml      Deploys k8s-observability-stack
projects/
  homelab-project.yaml     AppProject with source/destination restrictions
config/
  argocd-cm.yaml           ConfigMap customizations
  argocd-rbac-cm.yaml      RBAC for ArgoCD itself
```

## Why GitOps

Before ArgoCD, cluster state was maintained through a mix of YAML files, Helm releases, and `kubectl apply` runs from a workstation. The actual cluster state and the files in Git diverged over time — manual edits during incidents, forgotten test deployments, and Helm values that got applied but never committed.

GitOps solves this by making Git the single source of truth. Every change goes through a pull request. ArgoCD detects drift between the cluster and Git and either alerts or auto-remediates depending on the application's sync policy.

## App of Apps Pattern

The root `app-of-apps.yaml` is the only thing applied manually to the cluster. It points to the `apps/` directory, and ArgoCD discovers all other Application CRDs within. Adding a new application to the cluster means:

1. Add an Application YAML to the `apps/` directory
2. Commit and push
3. ArgoCD syncs the root app, discovers the new Application, and deploys it

Removing an application works the same way in reverse — with `prune: true`, deleting the Application YAML triggers ArgoCD to remove the deployed resources from the cluster.

## Bootstrap

```bash
# Install ArgoCD
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --version 7.x.x \
  -f install/argocd-values.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s

# Create the AppProject first (Applications reference it)
kubectl apply -f projects/homelab-project.yaml

# Apply config
kubectl apply -f config/

# Bootstrap the App of Apps — this is the last manual apply
kubectl apply -f apps/app-of-apps.yaml -n argocd
```

After the final command, ArgoCD takes over. All further application management is through Git.

## Accessing the ArgoCD UI

With the NodePort configuration, ArgoCD is accessible at:

```
http://192.168.1.110:30080
```

Retrieve the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

## Sync Policies

Different applications use different sync policies depending on their criticality and change frequency:

| Application | auto-sync | selfHeal | prune | Rationale |
|-------------|-----------|----------|-------|-----------|
| production-workloads | yes | yes | yes | Any drift from Git is an incident |
| monitoring-stack | yes | no | no | Manual interventions (silences, dashboard edits) should persist |

`selfHeal: true` means if someone runs `kubectl scale deployment --replicas=5` in production, ArgoCD will scale it back to whatever Git says within 3 minutes. This is intentional — the cluster should always reflect Git.

## AppProject Source Restrictions

The `homelab-project.yaml` restricts which repositories can deploy to the cluster. An Application cannot be created in the `homelab` project pointing at an arbitrary repository — only the repos explicitly listed are trusted. This prevents an Application manifest from being used to bootstrap malicious content from a third-party repo.

## Design Notes & Known Pitfalls

**selfHeal vs. incident response.** With selfHeal on, scaling a crash-looping deployment to 0 to stop the noise won't stick — ArgoCD scales it back. During an incident you disable selfHeal temporarily or annotate the app `argocd.argoproj.io/skip-reconcile=true` to pause reconciliation.

**Helm chart upgrades and ArgoCD.** ArgoCD tracks Helm chart versions in the Application source. Pinning `targetRevision` to a chart version prevents auto-upgrades; `targetRevision: "*"` upgrades whenever the chart repo updates, which can break things unexpectedly. Pinning to a specific version with intentional upgrades is the safer approach.

**App of Apps adds one level of indirection.** Debugging sync issues means checking both the root app and the child app — the root can show Synced while a child shows OutOfSync, so child apps must be checked individually.

**ArgoCD tracks resources by label.** Resources deployed by an Application get the `argocd.argoproj.io/app-name` annotation. Manually creating a resource ArgoCD also wants to manage causes ownership conflicts — ArgoCD should manage everything or nothing in a namespace.

## Related Repositories

- [k8s-production-patterns](https://github.com/Rekt-Dev/k8s-production-patterns) — what this deploys
- [k8s-observability-stack](https://github.com/Rekt-Dev/k8s-observability-stack) — monitoring stack managed by ArgoCD
- [k8s-security-hardening](https://github.com/Rekt-Dev/k8s-security-hardening) — RBAC and network policies that restrict ArgoCD's reach
