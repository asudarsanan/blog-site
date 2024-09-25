---
layout: post
title: "EKS Cluster Capacity Planning"
category: blogs
---

To plan the capacity for an Amazon EKS cluster that needs to run 100 pods, each with a 4GB memory footprint, we must take into account several factors, including node sizing, IP addresses, subnet configuration (public and private), and Kubernetes overhead. The steps below provide a comprehensive approach to plan both the networking and compute resources for your EKS cluster.

### 1. **High-Level Requirements**

- **100 Pods**: Each pod requires 4GB of memory.
- **EKS Overhead**: You’ll need to account for Kubernetes system pods (CoreDNS, Kube-proxy, etc.) in your resource allocation.
- **Public and Private Subnets**: Assume worker nodes are deployed in private subnets, with load balancers and internet-facing components in public subnets.

### 2. **Node and Pod Capacity Planning**

### Memory Considerations

Each pod has a 4GB memory footprint, so for 100 pods, the total memory requirement is:

- **Total memory requirement**: 100 pods * 4 GB = **400 GB of memory**.
- **Additional Kubernetes system memory**: Account for about 10-20% overhead for system components (CoreDNS, Kube-proxy, etc.). Assume 10% for simplicity:
    - System overhead = **40 GB**.
- **Total memory requirement** = **440 GB**.

### Choosing Node Instance Types

You will need to choose instance types that can accommodate the 4GB per pod requirement while leaving some buffer for node-level processes and overhead.

- **Instance Types**: Based on your requirement, instance types like `m5.2xlarge` or `m5.4xlarge` might be appropriate, as they provide balanced compute and memory. Here are some relevant details:
    - **m5.2xlarge**: 8 vCPUs, 32 GB of memory
    - **m5.4xlarge**: 16 vCPUs, 64 GB of memory

### Node Count Calculation

1. **Pods per Node**:
    - With `m5.2xlarge` (32 GB memory), the maximum number of 4GB pods per node = `32 / 4 = 8 pods`.
    - With `m5.4xlarge` (64 GB memory), the maximum number of 4GB pods per node = `64 / 4 = 16 pods`.
2. **Total Nodes Needed**:
    - To host 100 pods, you would need:
        - **With m5.2xlarge**: `100 / 8 = 13 nodes` (rounded up).
        - **With m5.4xlarge**: `100 / 16 = 7 nodes` (rounded up).

**Node Recommendation**: To strike a balance, you could choose `m5.4xlarge` nodes, requiring approximately **7 nodes** to handle 100 pods.

### CPU Considerations

Each pod will also consume CPU resources. If each pod requires 1 vCPU (assuming a typical load for applications requiring 4GB of memory), then:

- 100 pods * 1 vCPU = **100 vCPUs total**.

Each `m5.4xlarge` instance has 16 vCPUs, meaning the **total vCPUs** provided by the 7 nodes = 7 nodes * 16 vCPUs = **112 vCPUs**. This should be more than enough to handle the workload.

### 3. **IP Address Planning**

### ENI and IP Capacity

Each node will need multiple Elastic Network Interfaces (ENIs), and each ENI supports a specific number of secondary IPs based on the instance type.

- **m5.4xlarge** can support up to **4 ENIs**.
- Each ENI can support **30 IPs** (secondary addresses for pods).
- Total IPs per `m5.4xlarge` = **4 ENIs * 30 IPs = 120 IPs** per node.

You need to plan for both node IPs and pod IPs:

- 7 nodes * 120 IPs = **840 available IPs** (well beyond the 100 pod requirement).

### Subnet Sizing

You will need to create subnets with enough IP addresses for both your worker nodes and pods.

- **Subnet CIDR Calculation**: Assuming the nodes will be spread across three Availability Zones (AZs) for high availability:
    - For 7 nodes and their pods, you’ll need about 140 IPs per AZ (7 nodes * 120 IPs / 3 AZs ≈ 140 IPs per AZ).
    - A **/24 subnet** (256 IPs) in each AZ should suffice, allowing for growth (since a /24 subnet supports 256 IP addresses).

**Subnet Plan**:

- Public subnets for load balancers: `/24` (1 per AZ).
- Private subnets for worker nodes: `/24` (1 per AZ).

### 4. **Public and Private Subnet Configuration**

- **Private Subnets (Worker Nodes)**:
    - Place the EKS worker nodes in **private subnets** for security. These subnets will need access to the internet via **NAT gateways** for external communication (e.g., pulling container images).
    - Allocate a `/24` subnet per AZ for worker nodes.
- **Public Subnets (Load Balancers)**:
    - If you're using an **Application Load Balancer (ALB)** or **Network Load Balancer (NLB)** to expose services externally, place them in public subnets.
    - Allocate a `/24` subnet for each AZ for your load balancers.

### 5. **Load Balancer Planning**

- Use **Application Load Balancers (ALB)** in the public subnets to route traffic to your services.
- Ensure the **public subnets** are connected to the internet gateway for external access.
- **Private subnets** should route traffic through NAT gateways for outbound connections, such as image pulls from external repositories.

### 6. **EKS Control Plane and Networking**

- **EKS Control Plane**: This is managed by AWS, so it auto-scales based on your needs. No direct resource management is needed here.
- **VPC CNI Plugin**: Ensure that the VPC CNI plugin is properly configured to allocate IPs from the private subnets for pods. By default, the CNI will assign IPs from the node’s subnet.

### 7. **Autoscaling**

- **Cluster Autoscaler**: Configure the **Cluster Autoscaler** to automatically scale the number of worker nodes based on pod resource requests.
- Set the minimum and maximum node counts to ensure the cluster can scale up during traffic spikes and scale down to save costs during idle times.

Example configuration:

- Minimum node count: 3 (to ensure redundancy).
- Maximum node count: 10 (allowing room for extra capacity).

### 8. **Cost Considerations**

- **On-demand vs Spot Instances**: To optimize costs, consider a mix of **on-demand** and **spot instances** for your node groups. On-demand instances provide stability, while spot instances can offer savings if your workload can tolerate interruptions.
- You can create two separate node groups (one on-demand and one spot) and use taints and tolerations to assign specific workloads to each group.

---

### Capacity Planning Summary

| **Component** | **Details** |
| --- | --- |
| **Total Pods** | 100 Pods |
| **Memory per Pod** | 4GB |
| **Total Memory Requirement** | 440GB (including 40GB for system overhead) |
| **Node Instance Type** | m5.4xlarge (16 vCPUs, 64GB RAM) |
| **Nodes Required** | 7 Nodes |
| **Subnet Size** | /24 per AZ (for both public and private subnets) |
| **Availability Zones** | 3 AZs |
| **IP Addresses per Node** | 120 IPs (4 ENIs * 30 IPs each) |
| **Total Node IP Requirement** | 7 Nodes * 120 IPs = 840 IPs |
| **Public Subnets** | /24 per AZ (for ALB/NLB) |
| **Private Subnets** | /24 per AZ (for Worker Nodes) |
| **Cluster Autoscaler** | Minimum: 3 nodes, Maximum: 10 nodes |
| **Load Balancers** | ALB/NLB in Public Subnets |
| **Cost Optimization** | Mix of On-demand and Spot Instances |

This plan provides a balanced and scalable setup for running 100 pods with 4GB memory each while ensuring redundancy, cost-efficiency, and future scalability in Amazon EKS.