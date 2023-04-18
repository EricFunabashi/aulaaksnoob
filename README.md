# Objective
This README file is a guide to create a Gitops flow, using Flux and Kustomize in a k8s cluster. 

# Introduction 
This repository will hold the configurations for flux v2 gitops.
For more information about Flux and GitOps methodology go to: https://fluxcd.io/docs/

# Requirements
* Install the flux cli: https://fluxcd.io/docs/installation/#install-the-flux-cli
* Install Kustomize: https://kubectl.docs.kubernetes.io/installation/kustomize/
* Install Azure CLI: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli
* Install the Kubernetes CLI (kubectl): https://kubernetes.io/docs/tasks/tools/

# Connect to AKS cluster and save the Kubeconfig
* Open Azure CLI
* Run the following commands: 
    ```
    1. Set the subscription as default: 
    - az account set --subscription <Subscription ID>
    2. Get the credentials to access the AKS cluster:
    - az aks get-credentials --resource-group <ResourceGroupName> --name <ClusterName> --admin
    3. Copy to .kube path folder the reffered kubeconfig file: 
    - cp kubeconfig /home/<username>/.kube/config
    ```

# Architectural considerations
As a standard approach we are storing flux cluster and *applicational* configurations in the same repository! 

# Namespace Creation

* Run the following "kubectl apply" command, in order to create all the needed namespaces:

```
# kubectl apply -f ./base/namespaces.yaml
```

# Flux install
Perform the following steps:
* Go to your local folder holding the local copy of the intended flux repository. Git clone it, if needed;
* Export manifest to the .yaml file: 
```
# flux install --export > ./clusters/my-cluster/flux-system/gotk-components.yaml
```
* Apply the manifests on your cluster: 
```
# kubectl apply -f ./clusters/my-cluster/flux-system/gotk-components.yaml
```
* Commit and push the manifest to the master branch: 
```
# git add . && git commit -m "Starting Flux Installation" && git push
```
* Verify that the controllers have started: 
```
flux check 
```
Note: Output should be:
```
  ✔ Kubernetes 1.23.8 >=1.20.6-0
  ► checking controllers
  ✔ helm-controller: deployment ready
  ► ghcr.io/fluxcd/helm-controller:v0.24.0
  ✔ kustomize-controller: deployment ready
  ► ghcr.io/fluxcd/kustomize-controller:v0.28.0
  ✔ notification-controller: deployment ready
  ► ghcr.io/fluxcd/notification-controller:v0.26.0
  ✔ source-controller: deployment ready
  ► ghcr.io/fluxcd/source-controller:v0.29.0
  ► checking crds
  ✔ alerts.notification.toolkit.fluxcd.io/v1beta1
  ✔ buckets.source.toolkit.fluxcd.io/v1beta2
  ✔ gitrepositories.source.toolkit.fluxcd.io/v1beta2
  ✔ helmcharts.source.toolkit.fluxcd.io/v1beta2
  ✔ helmreleases.helm.toolkit.fluxcd.io/v2beta1
  ✔ helmrepositories.source.toolkit.fluxcd.io/v1beta2
  ✔ kustomizations.kustomize.toolkit.fluxcd.io/v1beta2
  ✔ ocirepositories.source.toolkit.fluxcd.io/v1beta2
  ✔ providers.notification.toolkit.fluxcd.io/v1beta1
  ✔ receivers.notification.toolkit.fluxcd.io/v1beta1
  ✔ all checks passed
```
* Create a SSH Key for the user to "tied" to this flux setup; Go to https://docs.microsoft.com/en-us/azure/devops/repos/git/use-ssh-keys-to-authenticate?toc=%2Fazure%2Fdevops%2Forganizations%2Ftoc.json&bc=%2Fazure%2Fdevops%2Forganizations%2Fbreadcrumb%2Ftoc.json&view=azure-devops for more info;

* Create a GitRepository object on your cluster by specifying the SSH address of your repo:

**IMPORTANT** Copy the git clone URL on the Azure Devops repository and change the --url=ssh:// for the result of copy/paste. Remember to change ":v3/" URL parameter to "/v3/" for working good. Example:

```
--url=ssh://git@ssh.dev.azure.com:v3/rafajhribeiro/* - Wrong
--url=ssh://git@ssh.dev.azure.com/v3/rafajhribeiro/* - Correct
```

* Doing the Flux source configuration as showed below:

```
flux create source git flux-system --url=ssh://git@ssh.dev.azure.com/v3/MSBookinfoApp/Bookinfo%20App/K8S-LAB --branch=main --ssh-key-algorithm=rsa --private-key-file=/home/<user>/.ssh/id_rsa --interval=1m
```

**IMPORTANT**: The above command will prompt you to add a deploy key to your repository, but Azure DevOps does not support repository or org-specific deploy keys. You may add the deploy key to a user’s personal SSH keys, but take note that revoking the user’s access to the repository will also revoke Flux’s access. The better alternative is to create a machine-user whose sole purpose is to store credentials for automation. Using a machine-user also has the benefit of being able to be read-only or restricted to specific repositories if this is needed.

The expected output is something like (answer "y" when prompted: `Have you added the deploy key to your repository:`):
```
  ✚ generating GitRepository source
✔ collected public key from SSH server:
  [...]
  ✚ deploy key: [...]

  Have you added the deploy key to your repository: y
► applying secret with repository credentials
✔ authentication configured
► applying GitRepository source
✔ GitRepository source created
◎ waiting for GitRepository source reconciliation
✔ GitRepository source reconciliation completed
✔ fetched revision: main/<ID review>
```
* Create a Kustomization object in your cluster:
```
# flux create kustomization flux-system --source=flux-system --path="./clusters/my-cluster" --prune=true --interval=1m
```
Note: "--prune=true" means that if anything is removed from the repo, will be removed from the cluster too

```
✚ generating Kustomization
► applying Kustomization
✔ Kustomization created
◎ waiting for Kustomization reconciliation
✔ Kustomization flux-system is ready
✔ applied revision main/<ID review>
```

* Go to the ./clusters/my-cluster, locate the root kustomization.yaml file and uncomment the line that was respective to flux directory, like this:

./cluster/my-cluster/kustomization.yaml
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- flux-system
```

* Go to the ./clusters/my-cluster/flux-system, locate the kustomization.yaml file and uncomment these lines that was respective to flux sync files, like this:

./clusters/my-cluster/flux-system/kustomization.yaml
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- gotk-components.yaml
- gotk-sync.yaml
```

* Export both objects, generate a `kustomization.yaml`, commit and push the manifests to Git:

```
  1. flux export source git flux-system > ./clusters/my-cluster/flux-system/gotk-sync.yaml

  2. flux export kustomization flux-system >> ./clusters/my-cluster/flux-system/gotk-sync.yaml

  3. git add . && git commit -m "Add Flux Sync Manifests into Cluster" && git push
```

* Wait for Flux to reconcile your previous commit with:

```
  # flux get kustomizations --watch
NAME            REVISION                SUSPENDED       READY   MESSAGE                              
flux-system     main@sha1:be981bf6      False           True    Applied revision: main@sha1:be981bf6
flux-system     main@sha1:be981bf6      False   Unknown Reconciliation in progress
flux-system     main@sha1:be981bf6      False   Unknown Reconciliation in progress
flux-system     main@sha1:be981bf6      False   Unknown Reconciliation in progress
flux-system     main@sha1:be981bf6      False   True    Applied revision: main@sha1:be981bf6
```

* Flux check

```
# flux get all
NAME                            REVISION                SUSPENDED       READY   MESSAGE                                           
gitrepository/flux-system       main@sha1:be981bf6      False           True    stored artifact for revision 'main@sha1:be981bf6'

NAME                            REVISION                SUSPENDED       READY   MESSAGE                              
kustomization/flux-system       main@sha1:be981bf6      False           True    Applied revision: main@sha1:be981bf6
```

# Adding Flux kustomizations for environments
As we have multiple environments and therefore will use a folder layout where shared configurations are stored in a `base` folder and the *differences* are in an `overlays` folder.

In the `overlays` folder, we will have a folder for each environment. This folder is going to be the *kustomization* target. So all (or almost all...) changes will be made in the `overlays` folder structure, **not in the `base`!** for security reasons. Because some wrong change will compromise all the cluster's environments.

* Create the overlay directory: 
Before to add the Kustomization, lets to create the overlay directory: 
```
mkdir -p ./overlays/my-cluster`
```

* Create the overlay directory: 
  * my-cluster:
    ```
    flux create kustomization <environment> \
    --source=flux-system \
    --path="./overlays/<environment>" \
    --prune=true \
    --interval=1m
    ```

* Export above kustomization and place them in the cluster config, by running the following command for kustomizations: 
```
# flux export kustomization 'dev' >> ./clusters/my-cluster/flux-system/gotk-sync.yaml
```
* Check the existing layout inside of `overlays/my-cluster`! There you will see a `kustomization.yaml` like:
```
  bases:
  - ../../base
  patches:
  - path: <file>.yaml
    target:
    kind: Ingress # Type of manifest, can be Deployment, CronJob, Ingress, etc
    name: <app_name>
```
  This yaml file is composed of two main sections:
  * bases: the shared code base for the configuration;
  * patches: the differences to the base, that depend on the environment;

* Commit and push the changes to repository;
```
# git add . && git commit -m "Created overlay to Dev environment" && git push
```

* As an example, here is the kustomizations list:

```
# flux get all
NAME                            REVISION                SUSPENDED       READY   MESSAGE                                           
gitrepository/flux-system       main@sha1:be981bf6      False           True    stored artifact for revision 'main@sha1:be981bf6'

NAME                            REVISION                SUSPENDED       READY   MESSAGE                              
kustomization/dev               main@sha1:be981bf6      False           True    Applied revision: main@sha1:be981bf6
kustomization/flux-system       main@sha1:be981bf6      False           True    Applied revision: main@sha1:be981bf6
```
**SETUP GITOPS DONE**