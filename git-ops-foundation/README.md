# GitOps Foundation
## Prerequisites for Local Setup
```
INFO: I've used Ubuntu terminal by enabling WSL inside my Windows.
```
1. For windows users [Linux/macOS users can skip this section]
2. Install Docker
   ```
   NOTE: Somehow If you use Docker Desktop and have WSL Integration enabled for your Ubuntu distro. Please disable it and continue with the local scratch installation of Docker inside the Ubuntu terminal
   ```
3. Install `kubectl` (The Kubernetes CLI)
   ```[bash]
   curl -LO "[https://dl.k8s.io/release/$](https://dl.k8s.io/release/$)(curl -L -s [https://dl.k8s.io/release/stable.txt](https://dl.k8s.io/release/stable.txt))/bin/linux/amd64/kubectl"
   
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   ```
4. Install `kind` (Kubernetes in Docker)
   ```[bash]
   [ $(uname -m) = x86_64 ] && curl -Lo ./kind [https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64](https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64)
   
   chmod +x ./kind
   
   sudo mv ./kind /usr/local/bin/kind
   ```
5. Install `helm` (The Kubernetes Package Manager)
   ```[bash]
   curl -fsSL -o get_helm.sh [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3)
   
   chmod 700 get_helm.sh
   
   ./get_helm.sh
    ```
## Local cluster setup
1. Create a file named `kind-config.yaml`
   ```[yaml]
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
    - role: control=plane
    - role: worker
    - role: worker
   ```
2. Create the cluster usging the above configuration file
   ```[bash]
   kind create cluster --name cka-cluster --config kind-config.yaml
   ```
3. Verify that the nodes are running
   ```[bash]
   kubectl get nodes
   ```

## Using Helm for application
Instead of writing raw YAML from sratch, we can use Helm templates for beginning
1. Create a new helm chart
   ```[bash]
   helm create my-webapp
   ```
2. Use the builtin `nginx` demo image to prove that the pipeline works
   - change `replicaCount: 1` to `replicaCount: 3` (to use 3 pods running via `kind`)

## GitOps Pipeline (ArgoCD)
Now, we install the brain of our GitOps operation: ArgoCD
1. Create a namespace and install ArgoCD
   ```[bash]
   kubectl create namespace argocd
    
   kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
2. Check the status of pods (once they are healthy and in running state)
   ```[bash]
   kubectl get pods -n argocd -w
   ```
3. Login the ArgoCD web UI by port forwarding to allow windows login to browser
   ```[bash]
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
   3.1. Fetch `Admin` password from a new WSL terminal
   ```[bash]
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
   ```
   *Login with Admin and the password just captured with the above command*

## Connecting the pipeline [github - argocd]
1. Create an argoCD app definition `argo-app.yaml`
   ```[yaml]
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
   name: my-gitops-app
   namespace: argocd
   spec:
   project: default
   source:
      repoURL: https://github.com/<YOUR-USERNAME>/<YOUR-REPO>.git
      path: <ROOT-OF-YOUR-REPO>/my-webapp
      targetRevision: HEAD
   destination:
      server: https://kubernetes.default.svc
      namespace: default
   syncPolicy:
      automated:
         prune: true
         selfHeal: true
   ```
2. Apply it to cluster
   ```[bash]
   kubectl apply -f argo-app.yaml
   ```
