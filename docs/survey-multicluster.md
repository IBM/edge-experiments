# Survey on multi-cluster projects 

This is a survey on existing Kubernetes multi-cluster management projects which might be relevant for edge scenarios.


Criteria | [Karmada](#karmada) | [Virtual Kubelet](#virtual-kubelet) | [Liqo](#liqo) | [Fleet](#fleet) | [ArgoCD](#argocd) | [OCM](#ocm) | [KCP](#KCP)
| --- | --- | --- | --- | --- | ---  | --- | --- |
Scalability | No data | Depends on provider | No data | [Findings](./published-scalability.md#fleet) | No data | [Findings](./published-scalability.md#ocm) | No data
Deployment Model | Push (pull planned) | Depends on provider | Push (pull planned) | Pull | Push | Pull | Push & Pull
GitOps-only |  No | No | No | Yes | Yes | No | No
Propagate generic k8s resources | No (only CM / Secrets referenced by pods) | No | No (only CM / Secrets referenced by pods) | Yes | Yes  | Yes | Yes
Clusters grouping | Yes | Depends on provider | No | Yes | No | Yes | Work in progress
Split deployment on multiple clusters | Yes | Depends on provider | Yes | No | No  | No | Yes
Manages cluster lifecycle | No | No | No | No | No | Yes (OCP only) | Yes

## Karmada

[Karmada](https://github.com/karmada-io/karmada) is an open source project created and maintained by Huawei. It runs as a separate control plane on a hosting Kubernetes cluster. The hosting cluster runs a separate API server, controller manager, scheduler and etcd in pods on the `karmada-system` namespace

### Features
* The scheduler is based on the Kubernetes scheduling framework for scheduling pods on remote clusters
* A Deployment may span multiple clusters
* Currently only the Push model is implemented, a Pull model is planned
* Currently propagates only pod, secrets and CMs

### Pros
* Separate the multi-cluster control plane from the host control plane
* Enable the creation of pods with standard Kubernetes deployments on remote cluster. 
* The only required additional API is the PropagationPolicy

### Cons
* The deployment is visible from the host, the pods on the managed clusters. There are no "proxy pods" so in order to see actual details of the pods you still need to access the managed clusters.
* When pods from same deployment are deployed on multiple clusters, and replicas = 1, the deployment ends up with more replicas than what specified in the desired state, which is confusing, as deployment was not designed to operate on a multi-cluster environment this way.

## Virtual Kubelet

[Virtual Kubelet](https://github.com/virtual-kubelet/virtual-kubelet
) is a CNCF Sandbox project. It provides a `Node` abstraction that enables running pods on non-Kubernetes environments (e.g. AWS Fargate). Virtual Kubelet alone does not provide multi-cluster orchestration capabilities, but is a foundational technology for projects providing “seamless scheduling” (Liqo, Admiralty, Tensil-kube). Additional components (e.g., scheduler, cluster registry) are required to support multi-cluster orchestration.
Virtual Kubelet defines a pluggable provider interface with an API that defines the actions of a typical Kubelet (e.g. CreatePod, GetPod, GetPodStatus, UpdatePod etc.). This approach enables implementing different backends for managing compute resources (e.g., a virtual Kubelet can represent a whole cluster. There are several providers available: Admiralty, Alibaba, Azure, AWS Fargate, OpenStack etc. Project contributors span a variety of organizations, the top 3 contributors are from Netflix and Microsoft

## Liqo
[Liqo](https://liqo.io) is a project created an maintained by the computer lab of Polytechnic of Turin University in Italy. It is based on the [Virtual Kubelet](#virtual-kubelet) project, and
because of that shares some common traits with other projects based on Virtual Kubelet such as
Admiralty and Tensile-kube.

### Design goals
- No disruption for k8s users
- API transparency: avoid introduction of new APIs so that no changes are required in
existing applications.
- Straightforward to operate for cluster admins
- No assumptions on Cluster config: any certified k8s distro
- Integrated approach between control plane and networking

Liqo introduces the abstraction of “Virtual Clusters". A virtual cluster is represented by a "Big Node", which provides the same API of standard Kubernetes nodes. This approach enables platform teams to perform operations such as drain clusters, cloud bursting and migrating apps without redeployment. Clusters registration and sharing of resources is based on policy-driven peering.  Each cluster can define which resources to share. Currently supported k8s distros include: K3S, Self-Hosted, GKE, AKS

## Fleet

[Fleet](https://fleet.rancher.io) enables GitOps-based distribution of applications to multiple clusters "at scale". It manages deployments from git of raw Kubernetes YAML, Helm charts, or Kustomize. Before deployment, all resources are dynamically turned into Helm charts Helm is used as the engine to deploy everything in the managed cluster. Fleet is open source with Apache 2 license, but it is mainly driven by by Rancher Labs & SUSE.

Currently Fleet is the only project among those surveyed that provides results of [large scale scalability experiments up to 1 M clusters](https://rancher.com/blog/2020/scaling-fleet-kubernetes-million-clusters) summarized [here](./published-scalability.md#fleet). Fleet was designed from the start with scalability and simplicity in mind. The install is relatively lightweight, it works on single and multi-cluster with the same API. Managed clusters install an agent and communicate with the Fleet Manager. Because it is agent-based, managed clusters can run in private networks and behind NATs, and with intermittent network connectivity. 

### Cluster Registration

Before deployment can happen, managed clusters are registered on the Fleet Manager
by generating a token on the fleet manager. The token, CA certificates,  
URL of the Fleet manager and optional labels are then provided during installation of the agent in each managed cluster. When the agent starts, it communicates with the Fleet Manager, which validates the token and if valid registers the cluster. 
The Fleet Manager creates a namespace for each managed cluster. This namespace
is used to sync content (spec & status) to that cluster.

### Multi-cluster workload deployment

- User provides a `GitRepo` CR which provides the location of a git repo with the app templates tp deploy to multiple clusters. `GitRepo` can also provide labels
selectors for targeting specific groups of clusters.
- The fleet manager starts a job (based on an embedded Tekton task) to pull down
the content of the git repo and package it into a single `Bundle` CR.
- The Bundle is then copied on each managed cluster namespace that was targeted by the label selector.
- Each managed cluster agent has a watch on bundles for its Fleet Manager namespace,
and sync (by pulling) the bundle to the managed cluster.
- The managed cluster agent extracts the content from the bundles and dynamically generates a helm chart, then it uses helm to deploy the content.
- Finally, the agent updates the status on the bundle in the Fleet Manager cluster namespace.

### Observations
Not all Kubernetes users have adopted GitOps so there is potential friction there.
On the other hand, Fleet comes bundled with Rancher for Rancher customers, so they still can use classic Rancher if not adopting GitOps.
For users who have adopted GitOps, they may have already adopted more popular GitOps technologies (Argo CD, Flux … ), and Fleet seems to aim to replace those solutions, so this could be another potential point of friction.

## ArgoCD

[ArgoCD](https://argoproj.github.io/argo-cd) is project focused on supporting  GitOps-style Kubernetes deployment of applications. It is currently a CNCF incubating project. It has a robust community and CNCF governance  It has some basic supports for multi-cluster deployment with the App of Apps pattern, however this approach works with a relatively small number of clusters, first of all because it is based on a Push model, and second
because the current design does not allow dynamic (e.g. label based) addressing
of target clusters, but it rather requires to list the API endpoint of each target
cluster where the app should be deployed. Furthermore, because of the push model, the Argo CD cluster must have network access to the API server of each targeted cluster, therefore it does not work if managed clusters are behind NATs or with intermittent network connectivity. In May 2020,  a new subproject
was started in the ArgoCD community to better address multi-cluster: the [ApplicationSet* controller and CRDs](https://github.com/argoproj-labs/applicationset). Although this provides a better UX for multi-cluster, it still relies on the existing ArgoCD application delivery push-based backend, thus it still suffers from the same issues observed in the app of apps. 

### Observations

ArgoCD provides an attractive approach to multi-cluster deployment for users
already adopting Argo. On the other hand it may still present the same friction
we discussed for Fleet for users not yet adopting GitOps or adopting different
GitOps solutions (e.g. Flux)


## OCM

[OCM - Open Cluster Management](https://open-cluster-management.io) is a community-driven project started by Red Hat, focused on multi-cluster and multi-cloud scenarios for Kubernetes apps. Open APIs are evolving within this project for cluster registration, work distribution, dynamic placement of policies and workloads, and much more.

OCM focuses on addressing the following requirements:

* Provide an API to determine the inventory of available clusters.
* Provide  a way to determine where to schedule and assign Kubernetes API manifests to a selected set of clusters.
* Provide a way to deliver desired Kubernetes API manifests to a selected set of clusters.
* Provide a way to govern how users access available clusters or groups of clusters in the fleet.
* Optionally, the service may need to extend the management agent with additional built-in controllers that should be run on managed clusters.

### OCM Architecture

The OCM architecture is modular and organized in two main groups:

#### Core Components
- Cluster manager: responsible for cluster registration and manifests distribution. 
- Klusterlet agent: agent running on the cluster managed by the hub.

#### Addons and Integrations
- Policy framework
    - Distribution and management of policies
    - Policy controllers: Plug-in controllers for different types of policies (e.g. IAM, Certificates etc.)
- Application lifecycle management: manage application resources on managed clusters
- Cluster lifecycle management
    - create, destroy of OCP clusters through the hive operator  
    - manages the addon lifecycle and a UI to give a better user experience on that process.

[Red Hat Advanced Cluster Management](https://www.redhat.com/en/technologies/management/advanced-cluster-management) is the commercial distribution of OCM. 
The ACM team has published their [scalability findings](https://www.youtube.com/watch?v=-vijgiXf6rc) summarized [here](./published-scalability.md#ocm) when scaling up to 1 K managed clusters. Note that the scalability of OCM and Fleet cannot be directly compared, as Fleet is only providing GitOps at scale, while OCM provides many different features. A fairer comparison might be comparing the scalability of
OCM and the scalability of [Rancher](https://rancher.com/products/rancher/).

## KCP

KCP is a project created by Red Hat, announced and open sourced at KubeCon EU in May 2021. The key idea of KCP is to offer a pure Kubernetes control plane separated
from containers orchestration. Therefore KCP provides a `minimal Kubernetes API server`, a Kubernetes without pods, nodes, services, but with support for CRDs, and a handful of useful abstraction such as Namespaces, Service Accounts, RBAC, Secrets, ConfigMaps etc. 

KCP has been Built on the premise that Kubernetes is much more than containers orchestration, and it provides a flexible platform for declarative APIs of all types, by enabling reconciliation pattern common to Kubernetes controllers for building robust, expressive systems.

It should be noted that here is larger agreement in the Kubernetes community that this is the way Kubernetes should have been from the start:
E.g. in the panel “The Power of Control Planes and the Kubernetes Resource Management Model” in the Crossplane Community Day with Kelsey Hightower (Google), Brendan Burns (Microsoft), Joe Beda (VMware), Brian Grant (Google)  (https://www.youtube.com/watch?v=WGfYlssfIIk ) there was agreement that Kubernetes should have been architected this way from the start (separate control plane from pod orchestration function).

Although KCP main goal is to provide a minimal Kubernetes API server, one of the
scenarios currently targeted by the KCP community is multi-cluster management.
The KCP project provides a PoC of multi-cluster management based on a 
`cluster-controller`, which along with the `Cluster` CRD allows KCP to connect to other fully-featured Kubernetes clusters, and includes these components:
- syncer, which runs on Kubernetes clusters registered with the cluster-controller, and watches kcp for resources assigned to the cluster
- deployment-splitter, which demonstrates a controller that can split a Deployment object into multiple "virtual Deployment" objects across multiple clusters.
- crd-puller which demonstrates mirroring CRDs from a cluster back to KCP

Since the project is very recent, there are not yet experiments or studies on scalability.

