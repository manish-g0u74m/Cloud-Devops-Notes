# AWS EKS — Complete Guide
### Types → Cluster Creation (GUI + CLI) → Node Joining → Add-ons/Drivers → Cost Optimization

---

## 1. EKS Deployment Types (Control Plane Models)

| Type | Control Plane Location | Nodes Location | When to Use |
|---|---|---|---|
| **EKS Standard (Region)** | AWS-managed, multi-AZ | EC2 / Fargate in your VPC | Default choice, 95% of workloads |
| **EKS on Outposts** | AWS-managed (in-region) OR local on the Outposts rack | Outposts hardware in your DC | Ultra-low latency on-prem apps, but still want AWS-managed control plane |
| **EKS Anywhere (EKS-A)** | You run and manage it yourself | Your own infra (vSphere, bare metal, Nutanix, Snow) | Fully air-gapped / disconnected environments, no AWS dependency |
| **EKS Distro (EKS-D)** | N/A — it's the open-source Kubernetes build EKS itself is made from | Anywhere | Building a custom Kubernetes distribution with the same tested binaries AWS uses |
| **EKS Hybrid Nodes** | AWS-managed (in-region) | Your on-prem/edge servers join as real nodes | Extend an AWS-controlled cluster onto your own hardware (needs VPN/Direct Connect) |

**Practical rule of thumb:** unless you have a compliance/data-residency reason to run on-prem, always pick **EKS Standard**. Everything below focuses there, with a dedicated section on Hybrid Nodes for your "external node" question.

---

## 2. Compute/Node Options — Overview Before the Deep Dive

| Option | Who manages the EC2 instance? | Auto-scaling | Node visibility |
|---|---|---|---|
| **EKS Auto Mode** | AWS (Karpenter-based, off-cluster) | Fully automatic | Instances exist in your account but you don't manage them directly |
| **Managed Node Groups (MNG)** | AWS provisions, you configure | Manual (ASG) or via Cluster Autoscaler/Karpenter | Full visibility, `kubectl get nodes` shows everything |
| **Self-Managed Nodes** | You, 100% | You configure everything | Full visibility, full responsibility |
| **Fargate** | No EC2 at all (microVM per pod) | Automatic, per-pod | No "nodes" in the traditional sense |
| **Hybrid Nodes (External)** | You (physical/on-prem machine) | N/A (physical capacity) | Shows as a node, but physically outside AWS |

---

## 3. Cluster Creation — GUI (AWS Console) — Step by Step

### Step 1: Navigate
`AWS Console → search "EKS" → Amazon Elastic Kubernetes Service → Clusters → Create cluster`

### Step 2: Choose creation mode
You'll see two paths:
- **Quick configuration (Auto Mode)** — AWS picks sensible defaults, sets up networking/compute/storage/LB automatically.
- **Custom configuration (Standard)** — full manual control, described below.

### Step 3: Cluster basics (Custom path)
- **Name**: cluster identifier (immutable after creation)
- **Kubernetes version**: pick the latest supported unless you have compatibility constraints (staying current avoids the extended-support cost jump — see Section 6)
- **Cluster IAM role**: a role with `AmazonEKSClusterPolicy` attached. Create this role BEFORE starting, in IAM console, trust policy = `eks.amazonaws.com`

### Step 4: Networking
- **VPC**: choose an existing VPC (must span ≥2 AZs)
- **Subnets**: select subnets across multiple AZs — mix of public (for internet-facing LBs) and private (for nodes/pods, best practice)
- **Security groups**: additional SG for control-plane-to-node communication (optional, EKS creates a default one)
- **Cluster endpoint access**:
  - *Public* — API server reachable from internet (simplest, least secure)
  - *Private* — only reachable from inside VPC (most secure, needs VPN/bastion/Cloud9 to manage)
  - *Public + Private* — public for convenience, but you can restrict by CIDR allowlist (recommended middle ground)

### Step 5: Observability
- Enable **Control Plane Logging**: API server, Audit, Authenticator, Controller Manager, Scheduler → sent to CloudWatch Logs (each log type has its own toggle — only enable what you'll actually use, since CloudWatch ingestion has a real cost)

### Step 6: Secrets encryption (optional but recommended)
- Enable **envelope encryption** using a KMS key for Kubernetes Secrets at rest

### Step 7: Add-ons selection screen
Console will prompt you to select add-ons to install alongside cluster creation:
- Amazon VPC CNI (pre-checked)
- CoreDNS (pre-checked)
- kube-proxy (pre-checked)
- Amazon EBS CSI Driver (NOT pre-checked in standard mode — check this box, it's the #1 thing people forget)
- Amazon EFS CSI Driver (only if you need shared storage)

### Step 8: Review and Create
Cluster provisioning takes **~10-15 minutes**. Status moves from `CREATING` → `ACTIVE`.

### Step 9 (separate, critical step): Add Compute
Once cluster is `ACTIVE`, go to the **Compute** tab of the cluster and click **Add Node Group** (or **Add Fargate Profile**). This is a completely separate action from cluster creation — a fresh cluster has **zero compute** attached. In this screen:
- **Node IAM Role**: needs `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`, `AmazonEKS_CNI_Policy` attached (create this role in IAM first)
- **AMI type**: Amazon Linux 2023, Bottlerocket, Windows, or custom via launch template
- **Capacity type**: On-Demand or Spot
- **Instance types**: pick specific families (e.g., `t3.medium`, `m5.large`)
- **Disk size**: root volume size in GiB
- **Scaling config**: Desired / Min / Max node count
- **Subnets**: usually private subnets

### Step 10: Connect kubectl to your new cluster (still needs CLI)
Even with GUI-created clusters, you must run this locally:
```bash
aws eks update-kubeconfig --name my-cluster --region ap-south-1
kubectl get nodes
```

---

## 4. Cluster Creation — Command Line (3 Methods)

### Method A: `eksctl` (fastest — recommended for most people)

Install:
```bash
# macOS
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
# Linux
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

**Simplest possible cluster + managed node group, one command:**
```bash
eksctl create cluster \
  --name my-cluster \
  --region ap-south-1 \
  --version 1.31 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 --nodes-min 1 --nodes-max 5 \
  --managed
```
This single command handles: VPC creation (if none given), IAM roles, control plane, node group, and kubeconfig wiring — **automatically**.

**Fargate-only cluster:**
```bash
eksctl create cluster --name my-cluster --region ap-south-1 --fargate
```

**Declarative config file (production best practice — repeatable, reviewable, Git-friendly):**
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: ap-south-1
  version: "1.31"

vpc:
  cidr: "10.0.0.0/16"
  nat:
    gateway: Single          # cost optimization — see Section 6

managedNodeGroups:
  - name: ng-ondemand
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 4
    privateNetworking: true
    labels: { role: general }

  - name: ng-spot
    instancesDistribution:
      instanceTypes: ["t3.medium", "t3a.medium", "t2.medium"]
      onDemandBaseCapacity: 0
      onDemandPercentageAboveBaseCapacity: 0   # 100% Spot
      spotAllocationStrategy: capacity-optimized
    desiredCapacity: 3
    minSize: 0
    maxSize: 10
    labels: { role: batch }
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
```
```bash
eksctl create cluster -f cluster.yaml
```

### Method B: Raw AWS CLI (manual — best for understanding what eksctl does under the hood)

```bash
# 1. Create the control plane
aws eks create-cluster \
  --name my-cluster \
  --role-arn arn:aws:iam::123456789012:role/eksClusterRole \
  --resources-vpc-config subnetIds=subnet-aaa,subnet-bbb,securityGroupIds=sg-xxx \
  --kubernetes-version 1.31

# 2. Wait for it to become ACTIVE
aws eks wait cluster-active --name my-cluster

# 3. Point kubectl at it
aws eks update-kubeconfig --name my-cluster --region ap-south-1

# 4. Create a managed node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name ng-1 \
  --node-role arn:aws:iam::123456789012:role/eksNodeRole \
  --subnets subnet-aaa subnet-bbb \
  --scaling-config minSize=1,maxSize=5,desiredSize=3 \
  --instance-types t3.medium \
  --capacity-type ON_DEMAND

# 5. Install required add-ons as EKS-managed add-ons
aws eks create-addon --cluster-name my-cluster --addon-name vpc-cni
aws eks create-addon --cluster-name my-cluster --addon-name coredns
aws eks create-addon --cluster-name my-cluster --addon-name kube-proxy
aws eks create-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver
```

### Method C: Terraform (Infrastructure-as-Code — best for teams/production)

Uses the community-maintained `terraform-aws-modules/eks/aws` module. Same underlying AWS API calls as the above, but state-tracked, version-controlled, and reviewable via pull requests. Recommended once you move past prototyping. (Happy to generate a full working `.tf` set if you want — just ask.)

---

## 5. Node Joining — How Each Type Actually Registers

This is the part people usually don't get explained clearly, so here's exactly what happens under the hood for each.

### 5.1 Managed Node Groups — Automatic Join
1. You give EKS an instance spec (type, AMI, min/max/desired count) via console/CLI
2. EKS creates an **Auto Scaling Group** and launches EC2 instances from an EKS-optimized AMI
3. On boot, the AMI's built-in `bootstrap.sh` script runs automatically — it fetches the cluster's API endpoint + CA certificate and configures `kubelet`
4. The node authenticates to the API server using its **IAM instance role**, which EKS automatically trusts (no manual `aws-auth` ConfigMap editing required — this is the "auto join")
5. Node appears in `kubectl get nodes` within 1-2 minutes
6. EKS also handles rolling updates, patching, and graceful node draining for you

### 5.2 Self-Managed Nodes — Manual Join
1. You launch your own EC2 instance (any AMI you choose)
2. You install `kubelet`, `containerd`/`docker`, and `aws-iam-authenticator` yourself
3. You manually run the bootstrap script, pointing it at the cluster name, API endpoint, and CA cert:
   ```bash
   /etc/eks/bootstrap.sh my-cluster \
     --apiserver-endpoint https://XXXX.eks.amazonaws.com \
     --b64-cluster-ca <base64-CA>
   ```
4. **Critical step**: register the instance's IAM role with the cluster so it's allowed to join. Two ways:
   - **Legacy**: edit the `aws-auth` ConfigMap in `kube-system` namespace, adding a `mapRoles` entry
   - **Modern (recommended)**: use **EKS Access Entries**:
     ```bash
     aws eks create-access-entry \
       --cluster-name my-cluster \
       --principal-arn arn:aws:iam::123456789012:role/my-node-role \
       --type EC2_LINUX
     ```
5. Node registers itself against the API server (`--register-node=true`) and appears in `kubectl get nodes`

### 5.3 Fargate — No Node Join at All
- No EC2 instance, no bootstrap script, no join process
- You define a **Fargate Profile** matching namespace + label selectors
- Pods matching the profile get scheduled onto per-pod isolated microVMs automatically
- CNI/kube-proxy/DaemonSets don't run the normal way on Fargate — networking is handled internally by AWS

### 5.4 EKS Auto Mode — Managed but Real EC2
- Karpenter-based system launches EC2 instances in your account automatically, in response to pending pods
- Uses immutable, AWS-patched Bottlerocket AMIs
- EBS CSI driver, Load Balancer Controller, and CNI are pre-wired **off-cluster** — you don't manage or see their pods the normal way
- Closest thing to "zero node management" while still using real EC2 economics

### 5.5 Hybrid / External Nodes — Physical Machines Outside AWS
This is what genuinely matches "external node":
1. Set up connectivity: **VPN or AWS Direct Connect** between your on-prem network and the cluster's VPC (mandatory — pods/nodes need routable IPs back to the control plane)
2. Set up credentials for the external machine to authenticate to AWS — either:
   - **SSM Hybrid Activation** (creates a managed-instance activation code/ID), or
   - **IAM Roles Anywhere** (X.509 certificate-based)
3. Install the **`nodeadm`** agent on the external server
4. Run:
   ```bash
   nodeadm init -c file://hybrid-node-config.yaml
   ```
   The config specifies cluster name, region, and the **remote pod/node CIDR** you've reserved for hybrid networking
5. Node joins exactly like an EC2 node from the Kubernetes API's perspective, but physically lives in your data center/edge site

**Note:** this is NOT zero-config auto-join — it requires explicit network peering and credential bootstrapping ahead of time. There's no way for a random external machine to "just join" without this setup (by design, for security).

---

## 6. Required Add-ons and Drivers (First-Time Setup Checklist)

| Add-on | What it does | Pre-installed by default? | Install command (if not) |
|---|---|---|---|
| **VPC CNI** (`amazon-vpc-cni-k8s`) | Assigns real VPC IPs to pods via ENIs — pod-to-pod networking | ✅ Yes | `aws eks create-addon --cluster-name my-cluster --addon-name vpc-cni` |
| **kube-proxy** | Implements Kubernetes Services via iptables/IPVS rules | ✅ Yes | `aws eks create-addon --cluster-name my-cluster --addon-name kube-proxy` |
| **CoreDNS** | Cluster-internal DNS resolution (service discovery) | ✅ Yes | `aws eks create-addon --cluster-name my-cluster --addon-name coredns` |
| **Amazon EBS CSI Driver** | Enables `PersistentVolumeClaim` backed by EBS (databases, StatefulSets) | ❌ No (unless Auto Mode) | `aws eks create-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver` |
| **Amazon EFS CSI Driver** | Shared/multi-attach NFS-based storage | ❌ No, optional | `aws eks create-addon --cluster-name my-cluster --addon-name aws-efs-csi-driver` |
| **AWS Load Balancer Controller** | Turns `Ingress` / `Service type=LoadBalancer` into real ALB/NLB | ❌ No (unless Auto Mode) | Helm install — see below |
| **Cluster Autoscaler** or **Karpenter** | Scales nodes based on pending pod demand | ❌ No | Helm install — see below |
| **Pod Identity Agent / IRSA (OIDC)** | Lets individual pods assume IAM roles securely | Needs OIDC provider set up on cluster creation | `eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve` |
| **Metrics Server** | Powers `kubectl top` and Horizontal Pod Autoscaler | ❌ No | Helm install — see below |

### Installing the AWS Load Balancer Controller (step by step)
```bash
# 1. Associate OIDC provider (needed for IRSA)
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# 2. Create the IAM policy AWS publishes for this controller
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 3. Create IAM role bound to a Kubernetes ServiceAccount (IRSA)
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# 4. Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Installing Karpenter (modern autoscaler, recommended over Cluster Autoscaler)
```bash
helm repo add karpenter oci://public.ecr.aws/karpenter/karpenter
helm install karpenter karpenter/karpenter \
  -n karpenter --create-namespace \
  --set settings.clusterName=my-cluster \
  --set settings.clusterEndpoint=$(aws eks describe-cluster --name my-cluster --query cluster.endpoint --output text)
```

### Installing Metrics Server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## 7. Cost Optimization — Detailed

### 7.1 Control plane cost (fixed, unavoidable, but manageable)
- Every cluster costs a flat **$0.10/hour (~$73/month)** while on a Kubernetes version in **standard support** (first 14 months after release)
- If you let a cluster's K8s version age past standard support into **extended support**, the fee jumps to **$0.60/hour (~$438/month)** — a **6x increase**
- **Action**: upgrade your cluster's Kubernetes version proactively, before the 14-month standard support window closes. This single habit avoids the most common "surprise EKS bill."
- Consolidate workloads into fewer clusters where reasonable (use namespaces + IAM/RBAC for isolation) rather than spinning up a cluster per team/environment unnecessarily — each cluster is a separate $73/month baseline.

### 7.2 Compute cost — choosing the right node type
| Node type | Extra fee on top of EC2? | Best for |
|---|---|---|
| Managed/Self-managed EC2 | None | Steady-state, predictable workloads — cheapest at scale |
| EKS Auto Mode | ~12% management premium on top of On-Demand EC2 rate (varies by instance; e.g., ~$0.0115/hr extra on an m5.large) | Teams that value zero ops over lowest possible bill |
| Fargate | Pay per vCPU/memory-second, no EC2 rate at all | Spiky, short-lived, low-traffic workloads |

**Important gotcha**: Fargate Spot does **not exist for EKS** (it's ECS-only). If you want Spot-level savings on EKS, you must use **Spot EC2 nodes**, not Fargate.

### 7.3 Spot Instances (up to 90% savings)
- Use Spot for stateless, interruptible, or batch workloads (CI/CD runners, queue workers, non-critical microservices)
- With Managed Node Groups, use `--capacity-type SPOT` or the `instancesDistribution` block in eksctl (shown in Section 4)
- With Karpenter or EKS Auto Mode, Spot interruption handling and consolidation is automatic
- Never run stateful, single-replica, or latency-sensitive workloads on pure Spot

### 7.4 Savings Plans and Reserved Instances (for predictable baseline load)
- **Compute Savings Plans**: up to 66% off, flexible across instance families/regions, applies to EC2 **and** Fargate
- **EC2 Instance Savings Plans**: up to 72% off, but locked to a specific instance family
- **Important exception**: Savings Plans discount the underlying EC2 rate, but do **NOT** discount the EKS Auto Mode management fee or the EKS control plane fee — those are always charged at standard rates regardless of commitment
- Strategy: put your steady-state minimum node count (the floor your ASG never scales below) on a Savings Plan, and let everything above that float on On-Demand or Spot

### 7.5 Right-sizing (the highest-impact lever most teams ignore)
- Overprovisioned CPU/memory `requests` are the single biggest source of EKS waste — a cluster can be billed at 100% utilization while actually running workloads at 20%
- Regularly compare Pod resource `requests`/`limits` against actual usage (via Metrics Server / CloudWatch Container Insights) and tune down
- Use **Karpenter's bin-packing/consolidation** — it actively repacks pods onto fewer, better-fitted nodes and terminates underutilized ones automatically (Cluster Autoscaler does not do this as aggressively)

### 7.6 Graviton (ARM) instances
- Roughly **10–20% better price-performance** than equivalent x86 instances for most containerized workloads
- No code changes needed for the vast majority of workloads (just ensure your container images support `arm64`, or use multi-arch images)
- EKS Auto Mode's default NodePool already mixes ARM + AMD; for MNG/self-managed you explicitly pick Graviton instance families (e.g., `m7g.large`)

### 7.7 Storage cost
- EBS volumes attached via the CSI driver are billed separately from compute — right-size PVC requests, and clean up orphaned/unattached volumes regularly (a common silent cost leak)
- Use `gp3` over `gp2` — cheaper baseline price with independently tunable IOPS/throughput

### 7.8 Networking cost traps
- **NAT Gateway**: charged per-hour PLUS per-GB processed. If nodes are spread across many AZs each with its own NAT Gateway, this multiplies. For cost-sensitive/dev environments, a **single NAT Gateway** (as shown in the eksctl config in Section 4) trades some AZ-redundancy for meaningfully lower cost — not recommended for critical production, but fine for staging/dev.
- **Cross-AZ data transfer**: pod-to-pod traffic across AZs is billed; where possible, use topology-aware routing/affinity to keep chatty services in the same AZ
- **Public IPv4 addresses**: now billed at $0.005/hr per address (since Feb 2024) — avoid assigning public IPs to nodes/pods that don't need direct internet exposure; route egress through NAT instead

### 7.9 Logging cost
- Control plane logs (API server, audit, etc.) sent to CloudWatch Logs incur ingestion + storage cost — only enable the log types you'll actually query/alert on, and set a retention policy (don't leave it at "never expire")

### 7.10 Quick cost-optimization checklist
- [ ] Cluster on a Kubernetes version within standard support (avoid the 6x extended-support fee)
- [ ] Spot instances for interruptible workloads
- [ ] Compute Savings Plan covering your steady-state floor capacity
- [ ] Karpenter (or Auto Mode) for bin-packing/consolidation instead of static-sized node groups
- [ ] Right-sized pod requests/limits, reviewed periodically against real usage
- [ ] Graviton where compatible
- [ ] `gp3` EBS volumes, orphaned volumes cleaned up
- [ ] Single NAT Gateway for non-critical environments
- [ ] CloudWatch log retention set explicitly, only necessary log types enabled

---

## 8. End-to-End First-Time Setup Order (Summary)

1. Create IAM roles: cluster role + node role
2. Create cluster (GUI or `eksctl create cluster`)
3. Wait for `ACTIVE` status
4. `aws eks update-kubeconfig` to connect `kubectl`
5. Add compute: Managed Node Group / Fargate Profile / Auto Mode
6. Associate OIDC provider (for IRSA)
7. Install EBS CSI driver
8. Install AWS Load Balancer Controller
9. Install Karpenter or Cluster Autoscaler
10. Install Metrics Server
11. Verify: `kubectl get nodes`, `kubectl get pods -A`
12. Apply cost-optimization checklist from Section 7
