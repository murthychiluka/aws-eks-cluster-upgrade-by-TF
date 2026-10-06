# EKS Cluster Upgrade Guide

## Step 1: Check Existing Version

```bash
aws eks describe-cluster \
--name project-eks \
--region us-east-1 \
--query "cluster.version"
```

Before going to upgrade we have to enable cordon to nodes to not deploy any pods on old node while doing version upgrade

```bash
kubectl cordon <old node>
kubectl cordon ip-10-0-3-8.ec2.internal
kubectl cordon ip-10-0-4-252.ec2.internal
```

## Step 2: Upgrade EKS Control Plane

Run:

```bash
aws eks update-cluster-version \
--region us-east-1 \
--name project-eks \
--kubernetes-version 1.32
```

## Step 3: Upgrade Node Groups

Once control plane upgrade completes, update worker nodes.

Option A — Rolling upgrade (recommended)

```bash
aws eks update-nodegroup-version \
  --cluster-name project-eks \
  --nodegroup-name eks-node-group \
  --kubernetes-version 1.32 \
  --region us-east-1
```

## Upgrade EKS Add-ons

Upgrade important components:

- CoreDNS
- kube-proxy
- VPC CNI

## Step 4: Upgrade Addons

Update each addon:

Example VPC CNI

```bash
aws eks update-addon \
--cluster-name project-eks \
--addon-name vpc-cni
```

Example CoreDNS

```bash
aws eks update-addon \
--cluster-name project-eks \
--addon-name coredns
```

Example kube-proxy

```bash
aws eks update-addon \
--cluster-name project-eks \
--addon-name kube-proxy
```

Example EBS CSI

```bash
aws eks update-addon \
--cluster-name project-eks \
--addon-name aws-ebs-csi-driver
```

Old node  
↓  
Drain  
↓  
Replace with new node  
↓  
Reschedule pods




## Resume Points – EKS Cluster Upgrade

- Planned and executed AWS EKS cluster version upgrades while minimizing downtime for production workloads.
- Managed node group upgrades using Terraform, ensuring worker nodes remained compatible with upgraded cluster versions.
- Implemented default EKS addons (CoreDNS, kube-proxy, VPC CNI, EBS CSI driver, pod identity) with automated upgrade paths using Terraform.
- Applied sequential addon deployment (depends_on) to reduce provisioning time and avoid AWS API throttling during upgrades.
- Leveraged Terraform variables and version control to dynamically manage cluster, node group, and addon versions for repeatable deployments.
- Ensured zero-impact upgrades by using rolling node updates, maintaining service availability during cluster version changes.
- Automated cluster and addon upgrades using Terraform best practices, including force_update_version and version pinning when needed.
- Monitored and validated post-upgrade cluster health, addon readiness, and application stability using AWS console and kubectl.


## Interview Explanation

"I have experience performing EKS cluster upgrades through multiple approaches. Manually, I've used the AWS console to upgrade the control plane and worker nodes while monitoring system components to ensure stability and zero downtime. Using the AWS CLI, I scripted cluster and node upgrades, validating addon and pod health to maintain operational reliability. With Terraform, I automated the entire upgrade process, managing cluster versions, node groups, and default addons like CoreDNS, kube-proxy, VPC CNI, and EBS CSI sequentially to avoid throttling. I also implemented best practices such as version pinning for critical addons, rolling updates for node groups, and health checks for system pods, ensuring upgrades are predictable and low-risk. By combining automation, monitoring, and sequential deployment, I was able to minimize downtime, maintain application availability, and deliver a fully auditable, repeatable process across multiple environments. This approach also improved team efficiency, reduced manual intervention, and allowed seamless scaling of the infrastructure."

*************************************************************************************************************************************************
Automated Kubernetes cluster and node pool upgrades using surge settings, automated node draining,
and Pod Disruption Budgets (PDBs) for controlled, highly available upgrades.
Please explain surge settings in detail

 Surge settings are especially important when you upgrade Kubernetes nodes because they help you replace old nodes without taking your applications down.
1. First, what happens during a node upgrade?

Suppose your cluster has:
Node 1 → 4 Pods
Node 2 → 4 Pods
Node 3 → 4 Pods

You want to upgrade the node OS or Kubernetes version.

Normally, Kubernetes needs to:

    Create a new/updated node
    Move workloads away from the old node
    Drain the old node
    Remove the old node

The question is:

    Where do the Pods go while the old node is being upgraded?

This is where surge helps.
2. What is a surge setting?

A surge setting allows the cluster to temporarily create extra nodes during an upgrade.

For example:
Before upgrade:

Node 1
Node 2
Node 3

Total = 3 nodes

If you configure:
maxSurge = 1

Kubernetes/cloud provider can temporarily create:
Node 1
Node 2
Node 3
Node 4  ← temporary new node

Now it can move workloads to Node 4 before removing one of the old nodes.

Then:
Old Node 1 → drained → removed

Node 2
Node 3
Node 4

Eventually, the cluster returns to the desired size of 3 nodes.
3. Why is this useful?

Without surge:
3 nodes
   ↓
Drain Node 1
   ↓
Only 2 nodes available
   ↓
Pods need to be rescheduled

There can potentially be resource pressure during this process.

With surge:
3 nodes
   ↓
Create temporary Node 4
   ↓
4 nodes available
   ↓
Move Pods
   ↓
Drain Node 1
   ↓
Remove Node 1
   ↓
Back to 3 nodes

So you have extra capacity during the upgrade.
# 4. Simple real-world example

Imagine an e-commerce application:
EKS Cluster

Node 1 → 10 Pods
Node 2 → 10 Pods
Node 3 → 10 Pods

Your application is serving customers.

You need to upgrade the node version.

If you use:
maxSurge = 1

the upgrade can temporarily become:
Node 1 → 10 Pods
Node 2 → 10 Pods
Node 3 → 10 Pods
Node 4 → new node

Pods from Node 1 can be rescheduled onto Node 4.

Then:
Node 1 → drained
Node 1 → removed

Node 2 → 10 Pods
Node 3 → 10 Pods
Node 4 → 10 Pods

Customers continue accessing the application.
# 5. Surge vs Pod Disruption Budget

These two concepts work nicely together, but they solve different problems.
Surge

Controls extra infrastructure capacity during the upgrade.
Surge
  ↓
Create extra node
  ↓
Move workloads
  ↓
Remove old node
Pod Disruption Budget (PDB)

Controls how many Pods can be voluntarily disrupted at once.

For example:

minAvailable: 2

means Kubernetes should maintain at least 2 available Pods during voluntary disruptions.

Suppose:
Pod A
Pod B
Pod C

During node draining:
Pod A → stays
Pod B → stays
Pod C → moved

The PDB helps prevent Kubernetes from disrupting too many Pods simultaneously.
6. Node draining

Drain means:

    "Stop scheduling new Pods on this node and safely evict the existing Pods."

For example:
Before:

Node 1
 ├── Pod A
 ├── Pod B
 └── Pod C

        ↓ drain

Node 1
 └── empty

        ↓

Node removed

The Pods are recreated/rescheduled on other available nodes.
7. How all three work together

This is the important part for your interview.
             Kubernetes Upgrade
                    │
                    ▼
              Surge Capacity
             Create extra node
                    │
                    ▼
              Drain old node
                    │
                    ▼
           PDB protects workloads
                    │
                    ▼
          Pods move to new nodes
                    │
                    ▼
             Remove old node
                    │
                    ▼
          Upgrade next node

So your resume statement:

    Automated Kubernetes cluster and node pool upgrades using surge settings, automated node draining, and Pod Disruption Budgets (PDBs) for controlled, highly available upgrades.

essentially means:

    "During upgrades, I ensured Kubernetes had enough temporary capacity, safely drained old nodes, and prevented too many application Pods from being disrupted at the same time."

One important interview point

The exact terminology/configuration differs slightly between EKS, AKS, and GKE. For example, AWS EKS managed node groups expose maxUnavailable/updateConfig, while AKS and GKE have their own surge-upgrade settings. So if an interviewer asks specifically about your implementation, describe the setting you actually configured rather than implying the same parameter was used on all three platforms.
