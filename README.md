# Kind, Helm, and Argo CD

This repository demonstrates GitOps on a local [kind](https://kind.sigs.k8s.io/) cluster:

1. `charts/parent` is packaged and published to Docker Hub as an OCI Helm chart. (We are doing this to learn how child charts can consume it)
2. `charts/child` depends on the parent chart and supplies its values.
3. Argo CD watches this Git repository and deploys the child chart to kind.

## Prerequisites

Install Docker, `kind`, `kubectl`, and Helm 3.8 or newer. Make sure Docker is running. You also need a Docker Hub account. A public parent chart and public Git repository need no pull credentials; private repositories and charts require credentials configured as described below.

## 1. Configure, package, and publish the parent chart

Replace `YOUR_DOCKERHUB_USERNAME` in `charts/child/Chart.yaml` with your Docker Hub username. The dependency repository is the OCI registry namespace; Helm appends `/parent` and the chart version when it pulls the dependency.

Log in to Docker Hub, package the parent chart, and push it:

```powershell
helm registry login registry-1.docker.io --username YOUR_DOCKERHUB_USERNAME
helm package charts/parent
helm push parent-0.1.0.tgz oci://registry-1.docker.io/YOUR_DOCKERHUB_USERNAME
```

The chart is published at `oci://registry-1.docker.io/YOUR_DOCKERHUB_USERNAME/parent:0.1.0`. Use a Docker Hub access token as the password if your account requires one.

## 2. Point the Argo CD Application to your Git repository

Edit `argocd/demo-app.yaml` and replace the placeholder `repoURL` with this repository's HTTPS or SSH clone URL. Commit and push the parent chart, child chart, and Application manifest. Argo CD must be able to read this Git repository.

The child pulls the parent through the `dependencies` entry in `charts/child/Chart.yaml`: it names the `parent` chart, pins version `0.1.0`, and points at the Docker Hub OCI namespace. First push that parent chart to Docker Hub. When Argo CD renders `charts/child`, its repo-server fetches the missing dependency from that registry automatically. No local `helm dependency update` is needed for this deployment. The parent chart is public in this setup, so Argo CD can pull it without Docker Hub credentials.

## 3. Create the kind cluster

From the repository root:

```powershell
kind create cluster --config kind/cluster-config.yaml
kubectl cluster-info --context kind-helm-argocd
```

The config creates a cluster named `helm-argocd` with one control-plane node and one worker. To remove it later, run `kind delete cluster --name helm-argocd`.

## 4. Install Argo CD

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=5m
```

## 5. Configure credentials if the Git repository or parent chart is private

The Git repository and the OCI Helm chart use separate credentials:

- **Private Git repository:** In Argo CD, open **Settings > Repositories** and connect the Git URL using a read-only Git token, GitHub App, or SSH deploy key. Argo CD uses this credential to clone `repoURL`.
- **Private parent chart in Docker Hub:** In Argo CD, configure OCI credentials for `registry-1.docker.io` under **Settings > Repositories**. Argo CD's repo-server uses them to pull the chart dependency. Public charts need no credentials.
## 6. Deploy the child chart with Argo CD

Apply the Application manifest:

```powershell
kubectl apply -f argocd/demo-app.yaml
kubectl get applications -n argocd
kubectl get all -n demo-app
```

Argo CD tracks `HEAD`, reads `charts/child`, and deploys the parent chart's resources in `demo-app`. The Application has automated sync, pruning, and self-healing enabled. Its `CreateNamespace=true` option creates `demo-app` automatically if it does not already exist.

## 7. Open the Argo CD UI

In one PowerShell window, forward the Argo CD server port:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open <https://localhost:8080> and sign in as `admin`. Retrieve the initial password in another PowerShell window:

```powershell
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($encoded))
```

## 8. Try the app and make changes

The child enables the Service, so forward the Service port to your machine:

```powershell
kubectl port-forward svc/demo-app-parent -n demo-app 8081:80
```

Visit <http://localhost:8081>. Change values in `charts/child/values.yaml` to configure the parent chart. To change the chart templates, edit `charts/parent`, publish a new parent chart version, update the dependency version in `charts/child/Chart.yaml`, and push the Git changes. Argo CD will reconcile the changes from Git.
