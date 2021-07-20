# Survey on Kubernetes projects for Edge 

This is a survey on Kubernetes-based projects targeting edge scenarios.


## KubeEdge

[KubeEdge](https://kubeedge.io/en/) is an open source project extending native containerized application orchestration and device management to hosts at the Edge.

TLDR: KubeEdge allows to extend a Kubernetes cluster with nodes specialized for Edge applications.

It is built upon Kubernetes and provides core infrastructure support for networking, application deployment and metadata synchronization between cloud and edge. It also supports MQTT and allows developers to author custom logic and enable resource constrained device communication at the Edge. KubeEdge consists of a cloud part and an edge part, both open sourced.

KubeEdge is composed of these components:

- Edged: an agent that runs on edge nodes and manages containerized applications.
- EdgeHub: a web socket client responsible for interacting with Cloud Service for edge computing (like Edge Controller as in the KubeEdge Architecture). This includes syncing cloud-side resource updates to the edge and reporting edge-side host and device status changes to the cloud.
- CloudHub: a web socket server responsible for watching changes at the cloud side, caching and sending messages to EdgeHub.
- EdgeController: an extended kubernetes controller which manages edge nodes and pods metadata so that the data can be targeted to a specific edge node.
- EventBus: an MQTT client to interact with MQTT servers (mosquitto), offering publish and subscribe capabilities to other components.
- DeviceTwin: responsible for storing device status and syncing device status to the cloud. It also provides query interfaces for applications.
- MetaManager: the message processor between edged and edgehub. It is also responsible for storing/retrieving metadata to/from a lightweight database (SQLite).

Some scalability findings have been published for KubeEdge in [this article](https://vmblog.com/archive/2021/04/16/managing-100-000-edge-nodes-on-china-s-highways-using-kubeedge.aspx) and are summarized [here](./published-scalability.md#kubeedge).



