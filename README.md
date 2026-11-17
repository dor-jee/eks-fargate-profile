# eks-fargate-profile

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

