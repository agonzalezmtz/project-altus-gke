# Project Altus: GKE Global Batch Engine

# Project Altus: GKE Global Batch Engine

**Project Altus** is a reference implementation of the **"Divide" architectural strategy** for Google Kubernetes Engine (GKE). [cite_start]It demonstrates how to orchestrate a distributed batch processing engine capable of executing **10,000 concurrent risk calculations** (financial simulations) in under **20 minutes**[cite: 200].

## 📋 Executive Summary

Legacy cluster architectures often hit physical limits when scaling burst workloads. Project Altus validates that by decoupling the "Control Plane" from the "Compute Plane" using GKE Enterprise, organizations can achieve:
* [cite_start]**Massive Scale:** 10,000 concurrent pods without saturating the Kubernetes API Server[cite: 203, 204].
* [cite_start]**70% Cost Reduction:** By leveraging Spot VMs in secondary regions for the heavy lifting.
* [cite_start]**Global Reach:** Bypassing regional IP quota limits and network bottlenecks[cite: 205].

## 🏗 Technical Architecture: Hub-and-Spoke Fleet

[cite_start]To overcome physical barriers like IP exhaustion and the "Thundering Herd" effect on the API Server, the infrastructure is divided into a **3-cluster GKE Fleet**[cite: 206].

![Architecture Diagram](https://via.placeholder.com/800x400?text=Architecture+Diagram+Placeholder)

| Cluster Role | Region | Type | Description |
| :--- | :--- | :--- | :--- |
| **Cluster-Hub** | `us-east1` | **GKE Autopilot** | Acts as the **"Brain"**. Hosts the Job Queue (Redis), Results Dashboard, and Observability stack. [cite_start]Stable control plane[cite: 207]. |
| **Cluster-Worker-A** | `us-central1` | **GKE Standard** | Acts as **"Muscle"**. [cite_start]Disposable cluster using **Spot VMs** for heavy compute[cite: 209]. |
| **Cluster-Worker-B** | `europe-west1` | **GKE Standard** | Acts as **"Muscle"**. [cite_start]Disposable cluster using **Spot VMs** for heavy compute[cite: 209]. |

### Key Technologies

* [cite_start]**Cloud Service Mesh:** Enables the 10,000 worker pods to connect securely to the Hub's Job Queue using internal DNS (`queue.prod.svc.cluster.local`) across regions without complex VPN tunnels[cite: 212].
* [cite_start]**Config Sync:** Pushes Kubernetes Job manifests to all worker clusters simultaneously, ensuring unified governance and drift detection.
* [cite_start]**Spot VMs:** Exclusively used in worker clusters to handle the burst workload, optimized with Node Termination Handlers[cite: 209, 275].
* [cite_start]**GKE Image Streaming:** Ensures nodes boot applications in under 4 seconds by streaming container data, avoiding network NAT saturation during the burst [cite: 287-289].

## 📂 Repository Structure

[cite_start]This repository follows the **Hierarchical Inheritance** pattern for Fleet management to manage complexity at scale[cite: 251]:

```text
├── system/               # Global policies (Security Agents, Logging) applied to ALL clusters [cite: 252]
├── fleets/
│   ├── hub/              # Configs for the Control Plane (Redis, Dashboards)
│   └── workers/          # Configs for Compute Planes (Spot VMs, Job Runners) [cite: 253]
├── namespaces/           # Namespace quotas and RBAC
└── workloads/            # The Batch Job Manifests


⚙️ Execution Flow

Fleet Connectivity: Cloud Service Mesh is enabled to create a unified trust domain, issuing mTLS certificates to every workload via Workload ID.


The Burst: A simple K8s Job manifest is pushed via Config Sync to the repository.


The Scale: 5,000 pods spin up in us-central1 and 5,000 in europe-west1 instantly, processing the queue in parallel.

🚀 Outcomes
Control Plane Isolation: The primary "Hub" cluster remained stable. The massive scheduling load of 10,000 pods was offloaded to the disposable "Worker" control planes.


Cost Efficiency: Shifting the burst workload to Spot VMs in cheaper regions resulted in a 70% cost reduction compared to running on reserved instances.


Resilience: The system successfully avoided the "Noisy Neighbor" effect and IP exhaustion common in single-cluster setups.

Project Altus is a reference implementation for "Navigating Large-Scale GKE".
