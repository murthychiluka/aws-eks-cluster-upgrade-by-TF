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

************************************
as of October 2026, EKS has Kubernetes 1.37 available, and AWS currently lists 1.34–1.37 in standard support and older versions such as 1.31–1.33 in extended support. AWS Documentation
# 1. What is an EKS version upgrade?
An EKS version upgrade means moving your Kubernetes cluster from one minor Kubernetes version to a newer supported version.
For example:
EKS 1.35
   ↓
EKS 1.36

or:
EKS 1.36
   ↓
EKS 1.37

It involves more than simply changing a version number. You need to consider:
- Control plane
- Worker nodes / managed node groups
- EKS add-ons
- Kubernetes APIs and deprecated resources
- Controllers such as AWS Load Balancer Controller
- Cluster Autoscaler / Karpenter
- CSI drivers
- kubectl
- Application compatibility
AWS specifically recommends checking upgrade insights and deprecated API usage before upgrading. AWS Documentation

# 2. What are the components involved in an EKS upgrade?
Think about an EKS cluster like this:
                    EKS Cluster
                        |
          +-------------+-------------+
          |                           |
    CONTROL PLANE                 DATA PLANE
    Managed by AWS                Managed by you
          |                           |
    API Server                 Managed Node Groups
    Scheduler                 Self-managed Nodes
    etcd                      Fargate
                                      |
                                    Pods
          |
    EKS Add-ons
    ├── VPC CNI
    ├── CoreDNS
    └── kube-proxy

During an upgrade, you normally consider:
# 1. Control Plane
AWS-managed:
API Server
Scheduler
Controller Manager
etcd

You initiate the Kubernetes version upgrade, but AWS performs the control-plane infrastructure update.
# 2. Worker Nodes
Examples:
Managed Node Group
Self-managed EC2 nodes
Fargate

These need to be brought to a compatible Kubernetes version.
# 3. EKS Add-ons
Important examples:
VPC CNI
CoreDNS
kube-proxy
EBS CSI Driver

AWS recommends updating the relevant add-ons after the control plane upgrade. AWS Documentation
# 4. Kubernetes applications/controllers
For example:
AWS Load Balancer Controller
Cluster Autoscaler
Karpenter
Metrics Server
Ingress controllers
CSI drivers

You need to verify that their versions support the new Kubernetes version.
5. Application manifests
You need to check for:
Deprecated APIs
Removed APIs
Deprecated CRDs
Old Helm charts
Old Kubernetes manifests

This is one of the major reasons an upgrade can fail even when the EKS infrastructure itself is healthy.
# 3. Control Plane vs Node Group upgrade
This is a very important interview distinction.
Control Plane Upgrade	Node Group Upgrade
Managed by AWS	Managed by you/AWS depending on node type
Upgrades Kubernetes API server/control-plane components	Upgrades worker node kubelet/AMI
Does not directly upgrade your EC2 nodes	Replaces/upgrades worker nodes
Initiated through EKS	Managed node groups can be updated through EKS
Happens first	Normally follows control-plane upgrade
Applications continue running during the control-plane update	Pods are gradually drained/rescheduled


Simple example
Current cluster:
Control Plane → 1.35

Node Group
├── Node 1 → 1.35
├── Node 2 → 1.35
└── Node 3 → 1.35

You want 1.36.
Step 1 — Control Plane
Control Plane → 1.36

Nodes → still 1.35

Then:
Step 2 — Node Group
Control Plane → 1.36

Node 1 → 1.36
Node 2 → 1.36
Node 3 → 1.36

AWS's documented upgrade flow is essentially control plane → nodes → add-ons/components. AWS Documentation
4. Is EKS control-plane upgrade automatic or manual?
The upgrade is initiated by you, but AWS performs the actual control-plane upgrade.
For example:
eksctl upgrade cluster \
  --name my-cluster \
  --version 1.36 \
  --approve

Or using AWS CLI:
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.36 \
  --region us-east-1

AWS then performs the control-plane replacement/rolling update. AWS Documentation
So in an interview, say:
"EKS control-plane upgrades are customer-initiated but AWS-managed. We select the target Kubernetes version, and EKS handles the underlying control-plane upgrade."

That's a good answer.
Important distinction
AWS does have automatic version movement in some lifecycle situations.
For example, if a cluster reaches the end of its extended-support period without being upgraded, EKS automatically moves it to the oldest currently supported version. 
So don't simply say:
"EKS upgrades automatically."

Instead say:
"Normal version upgrades are initiated by the customer; AWS manages the control-plane upgrade process. Automatic version movement can occur when a version reaches the end of its extended-support lifecycle."

5. What is the current Kubernetes version supported by EKS?
As of October 6, 2026, the latest Kubernetes version available on EKS is:
Kubernetes 1.37
AWS added EKS 1.37 on October 1, 2026. AWS Documentation
AWS currently lists:
Standard Support
----------------
1.37
1.36
1.35
1.34

Extended Support
----------------
1.33
1.32
1.31
``` :chatgpt-content-reference{index="7"}


For an interview, however, I'd phrase it carefully:

> **"The latest EKS version currently available is 1.37. EKS supports multiple Kubernetes versions simultaneously under standard and extended support."**

That is better than memorizing only one version because AWS releases new Kubernetes versions regularly.

---

# 6. Can we skip Kubernetes versions while upgrading?

### **No — for EKS control-plane upgrades, you should upgrade one minor version at a time.**

For example:

```text
1.35 → 1.36 → 1.37

You cannot normally do:
1.35 → 1.37

AWS explicitly states that EKS allows only one minor version at a time for control-plane upgrades. AWS Documentation
Example
Suppose your production cluster is:
1.34

and you want:
1.37

You need:
1.34
  ↓
1.35
  ↓
1.36
  ↓
1.37

And after each control-plane upgrade, you should bring the node groups/components into the appropriate supported state before proceeding.
7. One important interview nuance
You might hear someone say:
"Kubernetes supports version skew, so I can skip versions."

There is some truth here, but don't confuse version skew with skipping EKS control-plane upgrades.
For example:
Control Plane = 1.36
Node = 1.35

There are supported periods where nodes can temporarily be one or more minor versions behind the API server. AWS documents specific version-skew allowances. AWS Documentation
But that doesn't mean:
EKS Control Plane 1.35
        ↓
        1.37

is supported.
Think of these as two different concepts:
VERSION SKEW
     ↓
Temporary difference between
control plane and nodes

VERSION UPGRADE
     ↓
Moving EKS control plane
from 1.x → 1.x+1

⭐ Easy interview answer to remember
If the interviewer asks "Explain your EKS upgrade strategy", you can answer:
"We follow a controlled, one-minor-version-at-a-time upgrade strategy. First, we review EKS upgrade insights, deprecated APIs, add-on and controller compatibility. We upgrade the EKS control plane first, which is managed by AWS, then upgrade the managed node groups using a rolling/surge strategy with Pod Disruption Budgets to maintain availability. Finally, we update EKS add-ons such as VPC CNI, CoreDNS and kube-proxy, validate workloads, and perform post-upgrade health checks." AWS Documentation

One useful 2026-specific addition for your interviews: EKS now has version rollback, allowing a control-plane upgrade to be rolled back to the previous minor version within 7 days if issues are discovered; this is a newer capability announced by AWS in July 2026.
