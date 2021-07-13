# Published Scalability Findings

A number of projects such as [Fleet](https://fleet.rancher.io), [KubeEdge](https://kubeedge.io/en/) and [Red Hat ACM](https://www.redhat.com/en/technologies/management/advanced-cluster-management) have been focusing on management at scale of a fleet of clusters. This section summarizes key findings from these projects:

## Fleet

Findings on Fleet scalability experiments are described in [this article](https://rancher.com/blog/2020/scaling-fleet-kubernetes-million-clusters):

- **Scaling tests methodology**: It is just not feasible to test scalability of a very large number of devices ( 100 K - 1 M). Initially the tests focused on running a large number of agents on VMs with 384 GB of Memory (m5ad.24xlarge). Each VM ran 10 K3s clusters using k3d. Then those 10 K3s clusters ran 750 agents each, with each agent representing a downstream cluster. In total, each VM simulated 7,500 clusters. This approach was deemed too expensive and time consuming (it took 2 days to register 100 K simulated clusters). Therefore a new approach was adopted: running a single simulator that could make all the API calls that 1 million clusters would, without requiring a downstream Kubernetes cluster or deploying Kubernetes resources. Instead, the simulator made the API calls to register a new cluster, discover new deployments and report back status that they succeeded. Using this approach, it was possible to scale from zero to 1 million simulated clusters in a day.

- **Tuning Service Accounts and Rate Limiting**: since service accounts are generated for each new cluster, thus the rate at which   clusters could be registered is limited by the rate at which the controller-manager can create tokens for the service account. The 
solution was to modify the controller-manager’s default settings to increase the concurrency for service accounts creation(`–concurrent-serviceaccount-token-syncs=100`) and the overall number of API requests per second (`–kube-api-qps=10000`).

- **Etcd storage capacity limitations**: Etcd has a limit of 8GB for its keyspace, set to 2GB by default. The keyspace includes the current values and their previous values that have been yet to be garbage collected. A simple cluster object in Fleet will take about 6KB. For 1 million clusters, we will need at least 6GB. But a single cluster is typically about 10 Kubernetes objects, plus one object per deployment. So in practice, we are more likely to need more than 10x the space for 1 million clusters (60 GB). To
solve this issue the team used [Kine](https://github.com/rancher/kine) which allows to use Kubernetes with a traditional RDBMS.
In the test, the team has used Kine + RDS db.m5.24xlarge instances. By the end of the test, there were about 20 million objects in Kine. One issue left still unexplained: randomly a large list of objects (at most 10,000) would take 30 seconds to a minute (while
normally it would take 1-5 s).

- **Watch Cache Size**: When controllers load their caches, they first list all the objects and then start a watch from the revision of the list. If there is a very high change rate and listing takes long enough, you can easily get into a situation where you finish the list but can’t start a watch because the revision is not available in the API server watch cache or has been compacted in etcd. As a workaround, the team set the watch caches to a very high amount (`–default-watch-cache-size=10000000`)

- **Slow-to-Load Caches**: The standard implementation of Kubernetes controllers is to cache all objects you are working on in memory. With Fleet, that means the need to load millions of objects to build the caches. The listing of objects has a default pagination size of 500. It takes 2,000 API requests to load 1 million objects. If you assume we can make a list call, process the objects and start the next page every second, that means it would take about 30 minutes to load the cache. Increasing the page size to 10,000 objects did not help. With paging of 10,000 objects at a time, Kine would randomly take over a minute to return all the objects. Then the Kubernetes API server would cancel the request, causing the whole load operation to fail and have to be restarted. A workaround was increasing the API request timeout (`–request-timeout=30m`), but this is not an acceptable solution. Keeping the page size lower would ensure the requests were faster, but the number of requests increases the chances of one failing and causing the whole operation to restart. Restarting the Fleet controller would take up to 45 minutes. The restart time also applied to kube-apiserver and kube-controller-manager. Slow-to-Load Caches is still the biggest unsolved issue with Fleet scalability.

## KubeEdge

Findings on KubeEdge scalability based on production use with 100,000 edge nodes and 500,000 applications for ETC (Electronic Toll Collection) on [this article](https://vmblog.com/archive/2021/04/16/managing-100-000-edge-nodes-on-china-s-highways-using-kubeedge.aspx):

### Challenges:

- **Highly complex hardware management and application operations**:  Edge nodes in the ETC system were highly distributed, provided by multiple vendors, and span heterogeneous hardware architectures. Some edge industrial computers were so small they only had a 1 quad-core ARM SoC and 1 GB memory. Thus, the ETC solution needed to use as few resources as possible to manage edge nodes.
Additionally, the ETC system included six different management layers for applications, starting from the Highway Monitoring & Response Center out to the final road section. This could lead to complex Operations and Maintenance (O&M) and high costs. The ETC solution needed to address these issues by simplifying edge setup, minimizing the probability of problem occurrences, and reducing O&M costs.

- **Complex and unreliable network connectivity**: The network infrastructure varies significantly between each province. In some provinces, the network bandwidth is as low as 3 Mbit/s. The ETC solution must minimize the usage of the management network between the edge and cloud to fit within this low bandwidth. In addition, the network quality on some highways is very poor, which frequently causes network disconnections. The solution must have the ability to continue to operate offline in the case of network loss.


### Findings:

- **Multicluster Management**: Each province was managing its own nodes. One Kubernetes cluster was configured for each province, and a unified management layer was deployed to process complex cross-cluster data, allowing the central site to manage the edge side of each province. This indicates that there was not a single cluster with 100,000 edge nodes, but a set of clusters each managing
nodes for a province (**the article lists 29 provinces, thus an average of about 3450 edge nodes/cluster**).
Latest known reports for [Kubernetes 1.6 scalability](https://kubernetes.io/blog/2017/03/scalability-updates-in-kubernetes-1-6/) report scaling up to 5,000 node and 150,000 pod clusters, so it is conceivable that scaling up to 3500 nodes is feasible with
a recent standard Kubernetes distribution even without any special KubeEdge "magic dust". The main advantages
of KubeEdge seem in reducing resource resources footprint on the Edge, to run on 1 GB memory, on handling better
low quality network connectivity to the edge and providing support for connected device management on edge nodes.

- **Network partition at data center side causes errors when all edge nodes try to concurrently reconnect**:
After a network upgrade was performed at the Highway Monitoring & Response Center or a provincial center, all edge nodes in the provinces were disconnected from their Kubernetes clusters because of a network failure. After the network recovered, all edge nodes would simultaneously connect to the Kubernetes cluster at the central side. A large number of concurrent connections would impact the system at the central site and cause application exceptions. Therefore, *a dynamic back-off algorithm was used to relieve such impacts.*

- **Network bandwidth was insufficient to support status reporting**:
The infrastructure varies in each province. The network bandwidth in some provinces was as low as 3 Mbit/s. Therefore, the information reported by NodeStatus and PodStatus was simplified to reduce the impact of data reporting on the network.

- **Container images download optimizations**: the impact of a  large number of edge nodes and low bandwidth was reduced
adopting hierarchical container registry mirroring.

- **Pod Update issues**: The standard way to update Kubernetes applications in clouds is that the system deletes the pod of the old version, pulls the new image tag, and creates a new pod. This operation is acceptable when the cloud's network is in good condition. However, in this ETC edge environment, services will be interrupted for a long time, and the toll data will be lost. Therefore, the application upgrade process was also optimized for this project. To mitigate this issue, KebeEdge sends a notification to instruct the edge node to pre-download the image of the new version. After the image is downloaded, the system deletes the old pod and starts the new application. In this way, the overall service interruption duration during the upgrade was shortened from minutes to less than 10 seconds.

- **Edge Devices Failover**: Edge devices are deployed in active/standby mode, but they cannot communicate with each other through services like Elastic Load Balance (ELB), a service deployed only in the cloud. To prevent single point of failures (SPOFs), Keepalived was configured for containers on edge nodes.

## OCM

The use case under study required to scale up to 1 K clusters. Managed Clusters are OpenShift SNOs. Findings are summarized from this [ACM presentation](https://www.youtube.com/watch?v=-vijgiXf6rc) 

- **Namespace creation**: simple script to create namespaces. After creating about 2 K namespaces
the control plane crashes. The issue was due to the AWS gp2 volume type. As soon as the VM is provisioned,
there is a burst budget which provides good performance, but after that burst budget is exhausted the
IOPS drop dramatically. 

- **Controllers Crashing**: Controllers keep crashing with 1 K cluster (OOM). Increasing memory limits was
not considered a good solution, the team investigated the problem and found the issue was with the cache.
Watching a resource caches that resource. Get()/List() are always from the cache. The issue was that a lot
of unnecessary resources were also cached. 
**Lesson Learned:** do not cache objects you do not need ! 
    - use namespace scoped clients if possible
    - use labels on resources if possible
    - don't cache secrets for the whole cluster (controllers were crashing because of that, secrets were large and there are many !)
    - Starting with v0.9.0 controller-runtime allows `BuilderWithOption` to select objects to cache with labels
      or fields in the resource object (e.g. ,metadata.name)
Result: 10X memory reduction from 500+ MB to about 50 MB

- **Low Throughput**: this was observed for example when applying 1 K manifests (1 manifest for each cluster on 1 K cluster), it took ~ 40 s.
  Solution: tune `controller-runtime` settings:
  - increase client qps and burst (default 20 & 30) to 200 and 300. Optimized ~ 10 X
  - increase work queue rate limiter (default 10 qps and 100 bucket size)
  - MaxConcurrentReconciles - default is 1, but in some scenarios (long running tasks) it might help to increase concurrency
