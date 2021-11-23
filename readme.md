
# **EUTUXIA**
## Table of Contents
+ [About](#about)
+ [The Applications](#application)
+ [The Infrastructure](#infra)
+ [The Deployment](#deploy)
+ [Usage](#usage)

## About <a name = "about"></a>
Eutuxia defines the application resources, management, infrastructure and deployment of a proposed video sharing platform as declarative code.

## The Applications <a name = "application"></a>
The Applications—a frontend/user interface, a backend for video processing, and an in memory data store for caching, are packaged as containers to be deployed on Kubernetes.
The front and backend are both managed by deployment objects. The frontend uses *emptydir* volumes as she does not need to store data persistenly. However,  since her service needs to be accessible by Eutuxia users, it is exposed with a loadbalancer.
The backend does some heavy duty, Video transcoding and encoding are both data and CPU intensive workloads, so she uses a *persistentvolumeclaim*  with SSD for some scratch space, and GPU for horsepower. Only in-cluster traffic is expected by the backend, a ClusterIp manages this safely.
Our datastore's need for gracious restarts (access to the same volume pre pod failure) and a unique identity for each pod is why it is being managed by a statefulset.
A headless service provides DNS entry for the pods. This enables better communication with her target volume.
Prometheus and Grafana, installed with Helm, is tasked with observability for the infrastructure and applications.
[Istio](https://istio.io/) is used to power service to service communication. Envoy proxies are automatically injected as sidecars into our pods by labelling our name space with`istio-injection=enabled`.

## The Infrastructure<a name = "infra"></a>
Our applications will be managed in a Kubernetes cluster. Two buckets; an upload bucket where users can upload their videos, and a download bucket where videos already processed will be store for sharing.
Our infrastructure will be hosted on Google Cloud platform.
[Crossplane](https://crossplane.io/)  provides custom resource definitions that allow us provision our infrastructure as Kubernetes objects that we can manage, version control, maintain and reconcile. 

### Prerequisites

+ Mgnt-cluster (management cluster from where our operations will be performed)
+ A [Google Cloud Account](https://cloud.google.com/)
+ [gcloud](https://cloud.google.com/sdk/docs/install) sdk installed
+ [Helm](https://helm.sh/docs/intro/install/)

### Getting started with Crossplane
Crossplane can be [installed](https://artifacthub.io/packages/helm/crossplane/crossplane) using helm from the official repository.
After the installation, crossplane interacts with our cloud apis by installing the necessary cloud provider and allowing it the right permissions using a  [service account](https://cloud.google.com/iam/docs/creating-managing-service-accounts). 

### Usage
Runing `kubectl apply` against [eutuxia-infra.yaml](infra/eutuxia-infra.yaml) file, spins up a cluster with an extra GPU nodepool, and two buckets on GCP.
Each resource living as a k8s object that can be queried, modified, versioned, and managed by a reconcilatory loop, right in our cluster.

## The Deployment<a name = "deploy"></a>
Eutuxia's infrastructure being provisioned declaratively, as well as her application configurations, takes advantage of [GitOps](https://www.redhat.com/en/topics/devops/what-is-gitops/).
Using a Git as a single source of truth, Gitops uses pull requests to automate infrastructure and applications provisioning and deployment.
[ArgoCD](https://blog.argoproj.io/introducing-argo-cd-declarative-continuous-delivery-for-kubernetes-da2a73a780cd) is a custom resource definition that leverages K8s reconcilatory loops capabilities to sync the desired state — as specified in a git or helm repository, and the live state.

### Prerequisites
+ Mgnt-cluster
+ [Kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) for the workload cluster
+ [Git](https://git-scm.com/downloads)
+ [ArgoCD Cli](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
+ [Istio](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)

### Getting started with ArgoCD
ArgoCD can be [installed](https://artifacthub.io/packages/helm/argo/argo-cd) using helm. After [logging in](https://github.com/argoproj/argo-cd/blob/master/docs/getting_started.md), to use a remote repository like Github, ArgoCD access is [configured](https://cloud.redhat.com/blog/how-to-use-argocd-deployments-with-github-tokens) using the *argo-cm.yml* configmap.
Add a workload cluster using `argocd  cluster  add  CONTEXT  --name `

### ArgoCD Apps
ArgoCD Apps provide an isolated enviroment for applications and their configurations, from which it continously monitors the applications and compares the live state with the desired state as stated in the repo.
ArgoCD can monitor both git and helm repositories.

### Eutuxia Apps
+ [A Frontend](apps/frontend)
+ [A Backend](apps/backend)
+ [Redis Datastore](app/redis)
+ [Namespaces](app/namespaces)
+ [Prometheus](appset/promethus.yaml)
+ [Infra](infra)

The Frontend, Backend, Redis Datastore and Namespaces (aka in-house applications) are packaged as part of an [appset](https://blog.argoproj.io/introducing-the-applicationset-controller-for-argo-cd-982e28b62dc5).
Appset provides a template with autogenerates ArgoCD apps based on a number of rules. Here, the appset generates apps based on directory structure. This encourages flexibility in our team as it grows.
Prometheus being open sourced and installed from a helm repo, and Infra being owned by another team, are managed independently for proper separation of duties.
Finally, a superapp —bigapp, monitors all the apps.

### Usage 
Running `kubectl apply` against the [bigapp.yaml](bigapp/bigapp.yaml) file creates all the specified ArgoCD apps which installs all the applications in their respective clusters.

<img width="911" alt="Annotation 2021-11-21 134003" src="https://user-images.githubusercontent.com/78366369/143019931-952d9de1-1224-465e-9590-4b15af65c0dd.png">


<img width="898" alt="Annotation 2021-11-21 133412" src="https://user-images.githubusercontent.com/78366369/143019977-c68cc032-929f-4b6c-99c5-33a8de4c2121.png">


<img width="904" alt="Annotation 2021-11-21 133339" src="https://user-images.githubusercontent.com/78366369/143019997-dc6e12f3-b742-4731-87bd-c6ea02d888c5.png">
