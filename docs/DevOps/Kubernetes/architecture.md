---
sidebar_position: 1
---
# Kubernetes Architecture Overview
## What is Kubernetes?

Kubernetes (or K8s) is an open-source developed by Google for container management. It can deploy on vary platform (On-premise,  Cloud, Virtual Machine). Many large companies use it for system deployment and management.

## Why is Kubernetes?
We have several popular ways to deploy application

- Traditional

![Traditional](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/48vg6iwi3cdxswgr7ddd.png)

- Virtualized 

![Virtualized](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dtlhhro2yy8pfsiq2b92.png)

- Container

![Container](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ucxpja8qhxj0xac231wy.png)

Each approach has its advantages and disadvantages. Here, I will focus on the third approach (Container), which is more popular than the others. For a simple term, think of your server as a ship, and each container it carries is an isolated application ready to be deployed. Sound great, but we have another problem:

- How we manage it? 🤔
- How we scale it? 🤨
- What happen if the "ship" drown? 😱

![Drown](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pr3mi4cwbq31qzd5u9bn.png)

When we're anxious to find a solution, Google offers helpful solution - **Kubernetes** 😎

![K8s hero](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/atrnjjfvrh2vyd4ll9dx.png)

## How Kubernetes work?

Kubernetes enables application grouping and management via namespaces, and its Services facilitate traffic routing to the correct applications. When a "ship" drown, it move our container to another "ship" to ensure high availability for the app. And many, many things we can work on it. And here is the "secret" behind it - architecture

![K8s architecture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0m4mymxi7jxwq7la7cai.png)

## Control plane (Master node)

It responsible for application management and high availability (HA). We never run application on it.

**API Server**
- Kubernetes's heart
- All components must communicate through it, not directly.

**Control manager**
- It contain vary controller for each purpose (deployment, services, 
storage, etc.)

**Cloud control manager**
- Only appear on cloud platform

**Scheduler**
- Resource coordinator

**etcd**
- A database stores the cluster's state, including configurations, node data, pod data, and service data.

**P/s:** Backing up etcd is crucial for Kubernetes availability.


### Node (Worker node)

The application will run inside this; the number of worker nodes is limited only by your budget.


![Worker nodes](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7t6emwk0qrniivd2bmmm.png)

And worker node have 3 mains components:

**Container Runtime Interface (CRI)**
- Run container (docker, containerd or something else)
- A CRI contains one or more pods, each representing an application

**Kubelet**
- Receive and execute from API Server
- Transmit system information to API server
- Ensure containers/pods run correct state
- Continuously report pod/node status to the control plane and manage the pod lifecycle on the node

**Kubernetes Service Proxy**
- Maintain network rules for pod connectivity within and outside the cluster

## Conclusion
We have go through the Kubernetes architecture and know what is it. In [the next post](https://dev.to/locnguyenpv/deploy-kubernetes-on-premises-from-zero-4c2i), I will guide you install and run Kubernetes with on-premise system.

Happy Coding!

