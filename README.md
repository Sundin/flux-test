# flux-test

This is a guide for getting started with Kubernetes locally on MacOS!

The guide contains nothing groundbreaking, it is basically a summary of the official tutorials from [Flux](https://toolkit.fluxcd.io/get-started/) and [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) with some additional clarifications from me.

## Prerequisites

First we will install some tools. [Kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) is the Kubernetes CLI tool, [kind](https://kind.sigs.k8s.io/) is used for running a Kubernetes cluster locally on your own computer, and [Flux](https://toolkit.fluxcd.io/) is a tool for running [GitOps](https://www.gitops.tech/#what-is-gitops) – deploying changes to your cluster by making changes to a git repo.

    brew update
    brew install kind
    brew install kubectl
    brew install fluxcd/tap/flux

Optional tools. `watch` is used to watch commands and [stern](https://github.com/wercker/stern) is used for easily tail pod logs.

    brew install watch
    brew install stern

## Create a local Kubernetes cluster

We will use kind to set up a Kubernetes locally on your own computer.

Create a [Github personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) with all `repo` permissions enabled.

Set environment variables (for instance in `~/.bashrc` or `~/.zshrc`):

    export GITHUB_TOKEN=<your-token>
    export GITHUB_USER=<your-username>

Create your first cluster!

    kind create cluster

Verify that everything worked OK.

    kubectl cluster-info
    flux check --pre

Run the following command to create a repository that will contain the configuration for your Kubernetes cluster:

    flux bootstrap github \
        --owner=$GITHUB_USER \
        --repository=flux-test \
        --branch=master \
        --path=./clusters/my-cluster \
        --personal

## Add your first pod

Clone the repo the newly created repository.

We will now deploy a pod called [podinfo](https://github.com/stefanprodan/podinfo), which is a tiny sample web application made with Go, to our cluster. In order to do this, we will create a _GitRepository manifest_ (a file that defines that we will use a git repository as source for our pod) pointing to the podinfo repository's master branch. Other example of source types than git are [Helm](https://helm.sh/) repositories or buckets.

    flux create source git podinfo \
        --url=https://github.com/stefanprodan/podinfo \
        --branch=master \
        --interval=30s \
        --export > ./clusters/my-cluster/podinfo-source.yaml

We will also create a _Flux Kustomization manifest_ for the podinfo pod. The Kustomization API defines a pipeline for fetching, decrypting, building, validating and applying Kubernetes manifests, so this will configure Flux to build and apply the [kustomize directory](https://github.com/stefanprodan/podinfo/tree/master/kustomize) located in the podinfo repository.

    flux create kustomization podinfo \
        --source=podinfo \
        --path="./kustomize" \
        --prune=true \
        --validation=client \
        --interval=5m \
        --export > ./clusters/my-cluster/podinfo-kustomization.yaml

Commit and push the changes.

Watch Flux sync the application (this process is technically known as _[reconciliation](https://toolkit.fluxcd.io/core-concepts/#reconciliation)_):

    watch flux get kustomizations

When the synchronization finishes you can check that podinfo has been deployed on your cluster:

    watch kubectl -n default get deployments,services

_(Note: if you don't have watch installed, you can run the same commands without prefixing watch.)_

From this moment forward, any changes made to the podinfo Kubernetes manifest in the master branch will be synchronised with your cluster.

To follow logs in realtime for your pods, run:

    stern "podinfo-.*"

Alternatively, you can run `kubectl get pods` to figure out the name of one of your pods and then do `kubectl logs ${POD_NAME} ${CONTAINER_NAME}`.

## Access the service

At this point you might want to try out the actual service we now have up and running.

First of all we need to find out the name of some pod which we then can access. List all currently running pods with:

    kubectl get pods

Grab the name of any of the running podinfo pods (there should be two of them) and run the following command to forward port 8080 on your computer to port 9898 of the pod, which is were podinfo is listening for incoming traffic.

    kubectl port-forward <podname> 8080:9898

Access the podinfo service at http://localhost:8080/

## Ingress

By using the port-forward solution above, we will only get direct access to one of the two pods. A more robust solution is to use an Ingress, but this will require some more work. Step one is to set up a brand new cluster with an Ingress Controller installed.

### Recreate cluster

Delete the existing cluster:

    kind delete cluster

Create a kind cluster with `extraPortMappings` to allow the local host to make requests to the Ingress controller over ports 80/443 and `node-labels` to only allow the ingress controller to run on a specific node(s) matching the label selector.

    kind create cluster --config=kind/cluster-config.yaml

Add an NGINX Ingress Controller to the cluster:

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml

### Synch Flux to new cluster

Tell Flux to start synching to your new cluster instead (this is the same as the initial bootstrapping command we did, but without creating a new repository):

    flux install

    flux create source git flux-system \
        --url=https://github.com/$GITHUB_USER/flux-test \
        --branch=master \
        --interval=1m

    flux create kustomization flux-system \
        --source=flux-system \
        --path="./clusters/my-cluster" \
        --prune=true \
        --interval=10m

### Add ingress to pod

Now we need to clone the podinfo repo (or create a repo with a pod of our own). Change the GitRepository manifest to point to the new repo instead.

In the `kustomize` folder of the cloned repo, add a file called `ingress.yaml` with the following content:

    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
        name: example-ingress
    spec:
    rules:
        - http:
            paths:
            - path: /
                backend:
                serviceName: podinfo
                servicePort: 9898

And add this line to the bottom of `kustomize/kustomization.yaml`:

      - ingress.yaml

Push the changes.

    git push

Your service is now available at http://localhost:80/

## Advanced

If a Kubernetes manifest is removed from the podinfo repository, Flux will remove it from your cluster. If you delete a Kustomization from the flux-test repository, Flux will remove all Kubernetes objects that were previously applied from that Kustomization.

If you alter the podinfo deployment using `kubectl edit`, the changes will be reverted to match the state described in Git. When dealing with an incident, you can pause the reconciliation of a kustomization with `flux suspend kustomization <name>`. Once the debugging session is over, you can re-enable the reconciliation with `flux resume kustomization <name>`.

Since Kind eats up quite some CPU resources, it might make sense in pausing it when you don't need it. To pause the cluster, first run `docker ps` to get the container ID of your cluster. Then run `docker pause <containerId>`. To resume, run `docker unpause <containerId>` (who came up with that command name?!).
