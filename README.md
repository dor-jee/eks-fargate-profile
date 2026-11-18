# EKS Fargate Profile

![AWS](https://img.shields.io/badge/AWS-EKS-FF9900?logo=amazonaws&logoColor=white)
![Fargate](https://img.shields.io/badge/AWS-Fargate-FF9900?logo=amazonaws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.37-326CE5?logo=kubernetes&logoColor=white)
![Region](https://img.shields.io/badge/region-eu--central--2%20(Z%C3%BCrich)-informational)

Create an EKS cluster workload on AWS Fargate: a Fargate profile that sends selected pods to serverless compute while everything else keeps running on EC2.

Teams running stateless services on Kubernetes still spend time sizing, patching and scaling worker nodes for them. Fargate removes that layer: each pod gets its own isolated compute, and there is no node group to manage for it. Built as a practice project, this repository adds a Fargate profile to an existing EKS cluster so that pods in the `dev` namespace carrying the label `profile=fargate` run on Fargate, while `kube-system` and `default` stay on the EC2 node group.

Everything was configured explicitly through the AWS Management Console and `kubectl`, without IaC, so each setting and its tradeoff had to be understood rather than inherited from a module.

Sole engineer: designed, built and documented the setup; documentation drafted with AI assistance and checked against the build screenshots and the AWS documentation.

> **Looking for the node-group version?** This is the Fargate variant. The EC2 managed node group with the Kubernetes Cluster Autoscaler is documented in [eks-cluster-node-group](https://gitlab.com/cloudbuildops/eks-cluster-node-group).

**What you get:** an IAM pod execution role (`eks-fargate-role`), a Fargate profile (`dev-profile`) with a namespace and label selector, and an nginx Deployment that lands on its own Fargate node.

**Verified state:** the profile status is `Active` with 0 health issues, the nginx pod in `dev` is `1/1 Running` after 58 seconds, and `kubectl get node` lists a `fargate-ip-…` node as `Ready` next to the two EC2 nodes (see [Build walkthrough](#build-walkthrough)).

---

## Architecture

<img src="screenshots/00-architecture.png" alt="Architecture: EKS control plane with the dev-profile Fargate profile, EC2 node group in public subnets, Fargate node in a private subnet" width="800">

Editable source: [`screenshots/00-architecture.svg`](screenshots/00-architecture.svg)

---

## Tech stack

| Layer | Technology | Notes |
|---|---|---|
| Orchestration | Amazon EKS | cluster `eks-cluster-test`, nodes `v1.37.1` |
| Serverless compute | AWS Fargate | profile `dev-profile`, one pod per node |
| Existing compute | EC2 managed node group | 2 nodes, from the node-group project |
| Identity | IAM role `eks-fargate-role` | pod execution role, use case *EKS - Fargate pod* |
| Network | VPC with private subnets | pods take their IPs from the subnet range |
| Tooling | AWS Management Console, `kubectl` | no IaC |
| Region | `eu-central-2` (Zürich) | |

---

## Build walkthrough

Everything below was done in the AWS Management Console unless a command is shown.

### Prerequisites

- A running EKS cluster whose VPC has private subnets with NAT gateway access. The cluster `eks-cluster-test` and its VPC come from [eks-cluster-node-group](https://gitlab.com/cloudbuildops/eks-cluster-node-group).
- `kubectl` configured for that cluster
- Permission to create IAM roles and EKS Fargate profiles

### Repository contents

```bash
git clone https://gitlab.com/cloudbuildops/eks-fargate-profile.git
```

| File | Purpose |
|---|---|
| `nginx-config.yaml` | Sample nginx Deployment in namespace `dev` and a `LoadBalancer` Service, used to test the Fargate profile |


### 1. Create the IAM role for Fargate

Fargate needs a pod execution role. It allows access to the other AWS service resources that are required to run Amazon EKS pods on AWS Fargate.

1. Go to **IAM → Roles → Create role**.
2. Trusted entity type: **AWS service**.
3. Use case: **EKS**, then **EKS - Fargate pod**.
4. Role name: `eks-fargate-role`.

### 2. Create the Fargate profile

A Fargate profile is a rule that specifies which pods use Fargate when they are launched.

1. Go to **EKS**, select the cluster `eks-cluster-test` and open the **Compute** tab. Below the node groups you will see **Fargate profiles**.
2. Profile configuration:
   - Name: `dev-profile`
   - Pod execution role: `eks-fargate-role`, the role from step 1
   - Subnets: the private subnets of the cluster VPC. The VPC is required because Fargate pods get their IP addresses from your subnet IP range.
3. Pod selectors:
   - Namespace: `dev`
   - Match labels: key `profile`, value `fargate`

<img src="screenshots/01-fargate-pod-selection.png" alt="Configure pod selection: namespace dev, match label profile=fargate" width="800">

Pod selectors cannot be changed after the profile is created, so check them before the final step.

After creation the profile shows status `Active`, with the subnets and role ARNs on the Details tab:

<img src="screenshots/02-fargate-profile-details.png" alt="dev-profile details: Active, two subnets, pod execution role eks-fargate-role" width="800">

<img src="screenshots/03-fargate-profile-pod-selectors.png" alt="dev-profile Pod selectors tab: namespace dev, match label profile=fargate" width="800">
