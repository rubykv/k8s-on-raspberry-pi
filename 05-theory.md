# Kubernetes Theory

This section will cover the core concepts and theory behind Kubernetes architecture, components, and how they work together in a cluster.

## Core Components

- **Control Plane (Master)**: Manages the cluster state
- **Worker Nodes**: Run the applications
- **API Server**: The front-end for the Kubernetes control plane
- **etcd**: Consistent data store for cluster state
- **Scheduler**: Assigns work to nodes
- **Controller Manager**: Manages cluster controllers
- **kubelet**: Agent that runs on each node
- **kube-proxy**: Network proxy for service discovery

## Key Concepts

- **Pod**: Smallest deployable unit in Kubernetes
- **Service**: Abstract way to expose applications
- **Deployment**: Manages pod replicas and updates
- **Ingress**: Manages external access to services
- **Namespace**: Logical separation of resources

## Architecture Overview

Kubernetes follows a master-worker architecture where the control plane manages the cluster and worker nodes execute the workloads.

## Why This Matters for Raspberry Pi

Understanding these concepts is crucial when setting up Kubernetes on Raspberry Pi clusters, as each component needs to be carefully configured to work efficiently on ARM architecture with limited resources, especially when managing multiple nodes with constrained memory and storage. The ARM architecture and limited resources of Raspberry Pis require special attention to resource allocation, image selection, and component sizing compared to x86-based systems. Proper understanding helps in optimizing performance and avoiding common pitfalls when deploying Kubernetes on resource-constrained ARM devices like Raspberry Pi.

This theoretical foundation will guide the practical setup process in the following sections, ensuring that each step aligns with Kubernetes best practices and Raspberry Pi constraints, taking into account the unique challenges of ARM architecture and limited hardware resources such as memory, storage, and processing power required for efficient cluster operation, including considerations for network configuration, storage provisioning, and resource management across multiple Raspberry Pi nodes, while keeping in mind the need for lightweight components and efficient resource utilization for optimal performance on ARM-based Raspberry Pi hardware, with careful attention to container images optimized for ARM architecture and resource constraints, ensuring that all components are compatible with the Raspberry Pi's ARMv7 or ARM64 architecture and can operate within the limited memory and storage constraints of the devices.

### Services

#### What's NodePort
NodePort is one of Kubernetes’ simplest ways to expose a Service outside the cluster.

Think of it as:

“Open a fixed port on every node and forward traffic to my Pods.”

**One-line definition**

NodePort exposes a Kubernetes Service by opening the same port on all worker nodes, forwarding traffic to the Service, which then load-balances to Pods.

Browser / Client
        |
        |  http://<NODE_IP>:31234
        v
+--------------------+
| Worker Node (Pi)   |  <-- NodePort (31234)
|                    |
|   kube-proxy       |
|        |           |
+--------|-----------+
         v
   Service (ClusterIP)
         |
         v
     Pod A / Pod B


**Key point:**
➡️ ANY node IP works, even if the Pod is running on another node.
