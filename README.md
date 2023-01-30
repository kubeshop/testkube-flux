# Testing cloud native applications with Flux and Testkube

This repository sets up Flux to use Testkube to generate testing resource manifests that Flux will then apply to a Kubernetes cluster. This repository covers the steps to do that. 

## Manual Flux configuration

### 1. [Fork this repository](https://github.com/kubeshop/testkube-flux/fork) and clone it locally

```sh
git clone https://github.com/$GITHUB_USER/testkube-flux.git
```

### 2. Start a Kubernetes cluster

You can use Minikube, Kind or any managed cluster with a cloud provider (EKS, GKE, etc). In this example we're using [Kind](https://kind.sigs.k8s.io/). 

```sh 
kind create cluster
```

### 3. Create a Github Classic Token: 

Must be of type [__Classic__](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token#creating-a-personal-access-token-classic) (i.e. starts with `ghp_`)

```sh
GITHUB_TOKEN=<ghp_>
GITHUB_USER=<username>
```

And export the environment variables in your terminal.

### 4. Install Flux in the cluster and connect it to the repository 

Install the [Flux CLI](https://fluxcd.io/flux/installation/) and run: 

```sh
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=testkube-flux \
  --path=cluster \
  --personal
```

### 5. Create a Flux Source and a Kusktomize Controller

The following command will create Flux source to tell Flux to apply changes that are created in your repository: 

```sh
flux create source git testkube-tests \
  --url=https://github.com/$GITHUB_USER/testkube-flux \
  --branch=main \
  --interval=30s \
  --export > ./cluster/flux-system/sources/testkube-tests/test-source.yaml
```

And now create a Flux Kustomize Controller to apply the Testkube Test CRDs in the cluser using [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/): 


```sh 
flux create kustomization testkube-test \
  --target-namespace=testkube \
  --source=testkube-tests \
  --path="cluster/testkube" \
  --prune=true \
  --interval=30s \
  --export > ./cluster/flux-system/sources/testkube-tests/testkube-kustomization.yaml
```

### 6. Install Testkube in the cluster

Install the Testkube CLI from https://kubeshop.github.io/testkube/installing

And run the following command to install Testkube and its components in the cluster: 

```sh
testkube install 
```

### 7. Create a `Test CRD` with `testkube` CLI

In this example the test being used is a Postman test, which you can find in `/img/server/tests/postman-collection.json`. 

To create a Kubernetes CRD for the test, run:

```sh 
testkube generate tests-crds img/server/tests/postman-collection.json > cluster/testkube/server-postman-test.yaml
```

### 8. Add the generated test to the Kustomize file: 

The name of the test file created in the previous step is `server-postman-test.yaml`, add that to the Kustomize file located in [`cluster/testkube/kustomization.yaml`](./cluster/testkube/kustomization.yaml): 

```diff
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
+  - server-postman-test.yaml
```

### 9. Push all the changes to your repository: 

```sh 
git pull origin main
git add -A && git commit -m "Configure Testkube tests"
git push
```

### 10. Your tests should be applied in the cluster:

To see if Flux detected your changes run: 

```sh
flux get kustomizations --watch
```

And to ensure that the test has been created run: 

```sh 
testkube get test

  NAME                    | TYPE               | CREATED                       | LABELS                                            |
--------------------------+--------------------+-------------------------------+---------------------------------------------------+
  postman-collection-test | postman/collection | 2023-01-30 18:04:13 +0000 UTC | kustomize.toolkit.fluxcd.io/name=testkube-test,   |
                          |                    |                               | kustomize.toolkit.fluxcd.io/namespace=flux-system |

```