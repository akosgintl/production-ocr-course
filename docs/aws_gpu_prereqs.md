# GPU Deployment on AWS (EKS)

This guide takes you from zero to a GPU-ready EKS cluster for the OCR
pipeline. Same **asymmetric hardware** idea as the Azure/GCP guides: cheap
**T4** nodes for orchestration and layout extraction, a premium GPU pool for
the heavy VLM generation. Read this fully before spending money — AWS's GPU
lineup has one real surprise (Section 1) that the other two clouds don't.

---

## 1. Target instance types

> ⚠️ **AWS has no single-GPU A100 instance.** Azure's `NC24ads_A100_v4` and
> GCP's `a2-highgpu-1g` both give you **one** A100 per VM. AWS's A100 instance,
> `p4d.24xlarge`, only exists as a fixed **8-GPU, 96-vCPU** node — there is no
> smaller size. That breaks the "one GPU per autoscaled node" shape the other
> two guides use, so this course's AWS track uses **H100** instead, via the
> single-GPU `p5.4xlarge` (GA since August 2025), to keep the same 1-GPU/node
> autoscaling model. If you specifically need the A100 architecture to match
> Azure/GCP, `p4d.24xlarge` is documented as the alternative below — you just
> accept that one node gives you 8 GPUs at once, not 1.

| Role                          | Instance type    | GPU                            | vCPUs/VM |
|-------------------------------|-------------------|---------------------------------|----------|
| Layout workers + orchestrator | `g4dn.xlarge`     | 1× NVIDIA T4 (16 GB)            | 4        |
| VLM inference (recommended)   | `p5.4xlarge`      | 1× NVIDIA H100 (80 GB)          | 16       |
| VLM inference (A100 parity)   | `p4d.24xlarge`    | 8× NVIDIA A100 40 GB (fixed)    | 96       |

> **If you go the `p4d.24xlarge` route:** a single node already provides more
> GPUs (8) than the 4-GPU target the Azure/GCP guides request, so you'd only
> ever need **one** such node, not an autoscaled pool of four. Budget for it —
> at time of writing this is roughly $32–40/hr on-demand, billed whether 1 or
> 8 of its GPUs are in use.

AWS's other GPU families for context: `G4`/`G5`/`G6` (T4/A10G/L4, inference,
cheap), `P3`/`P4`/`P5` (V100/A100/H100, training-oriented but usable for
inference), no direct L40/L40S equivalent (closest is `G6e`, L40S-based).

---

## 2. Quota to request (per region)

AWS groups GPU instances into **families** and measures quota in **vCPUs**,
same mechanic as Azure (not GPU count, like GCP). The two families you need:

| Quota name                                    | Request  | Why                                                  |
|------------------------------------------------|----------|-------------------------------------------------------|
| `Running On-Demand G and VT instances`         | 16 vCPU  | 4 × `g4dn.xlarge` (4 vCPU) → 4 T4 GPUs                |
| `Running On-Demand P instances`                | 64 vCPU  | 4 × `p5.4xlarge` (16 vCPU) → 4 H100 GPUs              |

> Going the A100-parity route instead? Request **96 vCPU** on the `P` family
> (1 × `p4d.24xlarge`) rather than 64.
>
> Both quotas start at **0** on every new account — Free plan or Paid, it
> doesn't matter. This is not gated behind a trial/paid switch like
> Azure/GCP; it's just always 0 until you ask.

---

## 3. Install the AWS CLI, eksctl, and kubectl

**AWS CLI — macOS (Homebrew)**
```bash
brew install awscli
```

**AWS CLI — Linux**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
```

**AWS CLI — Windows**: installer at
https://awscli.amazonaws.com/AWSCLIV2.msi

**eksctl**
```bash
# macOS
brew install eksctl

# Linux
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp && sudo mv /tmp/eksctl /usr/local/bin

# Windows
choco install eksctl
```

**kubectl**: install per
https://kubernetes.io/docs/tasks/tools/ (or `brew install kubectl`).

**Configure and log in**
```bash
aws configure                       # paste your IAM access key + secret, pick a default region
aws sts get-caller-identity
```

---

## 4. Get GPU access, step by step

### 4.1 Open Service Quotas for EC2

1. In the console, search for **Service Quotas**.
2. Left menu → **AWS services** → search `Amazon Elastic Compute Cloud (Amazon EC2)`.

### 4.2 Request the T4 (G/VT) quota

1. In the quota search box type `G and VT`.
2. Open **Running On-Demand G and VT instances** — note the current limit
   (almost certainly `0`).
3. Click **Request increase at account level**, enter **16**, submit.

### 4.3 Request the H100/A100 (P) quota

1. Clear the search, type `P instances`.
2. Open **Running On-Demand P instances**.
3. **Request increase at account level**, enter **64** (or **96** if you're
   going the `p4d.24xlarge` A100-parity route), submit.

### 4.4 CLI equivalent (look up the quota code, then request)

Quota codes aren't worth memorizing — look them up dynamically each time:

```bash
REGION=eu-west-2

# Find the G/VT quota code and request 16 vCPU
GVT_CODE=$(aws service-quotas list-service-quotas --service-code ec2 --region $REGION \
  --query "Quotas[?contains(QuotaName, 'G and VT instances')].QuotaCode" --output text)
aws service-quotas request-service-quota-increase --service-code ec2 \
  --quota-code $GVT_CODE --desired-value 16 --region $REGION

# Find the P quota code and request 64 vCPU (or 96 for p4d)
P_CODE=$(aws service-quotas list-service-quotas --service-code ec2 --region $REGION \
  --query "Quotas[?contains(QuotaName, 'P instances')].QuotaCode" --output text)
aws service-quotas request-service-quota-increase --service-code ec2 \
  --quota-code $P_CODE --desired-value 64 --region $REGION
```

### 4.5 If the request is denied or stuck pending

This is common on new accounts, especially for the `P` family — it isn't an
error. Open a **Support case** (Service Quotas console has a direct link from
the request's status page) with a short justification:

> Deploying an OCR / document-intelligence pipeline on EKS in eu-west-1. The
> T4 (`g4dn.xlarge`) node pool handles layout extraction; the H100
> (`p5.4xlarge`) pool handles VLM inference. Both pools autoscale with
> scale-to-zero for cost efficiency. Requesting 16 vCPU for G/VT and 64 vCPU
> for P instances.

Request the two families as **separate cases** — G/VT usually clears fast; P
can take longer, especially for A100/H100 sizes.

---

## 5. Choose a region and verify availability

GPU instance availability varies **by region and by Availability Zone**, so
check the exact combination before committing.

**Scan a few regions for both instance types:**
```bash
# for REGION in eu-west-1 eu-central-1 us-east-1 us-east-2 us-west-2; do
for REGION in eu-central-1 eu-central-2 eu-west-1 eu-west-2 eu-west-3 eu-south-1 eu-south-2 eu-north-1; do
  echo "=== $REGION ==="
  aws ec2 describe-instance-type-offerings \
    --location-type region \
    --filters Name=instance-type,Values=g4dn.xlarge,p5.4xlarge \
    --region $REGION \
    --query "InstanceTypeOfferings[].InstanceType" --output text
done
```

**Then check per-AZ, for your chosen region:**
```bash
REGION=eu-west-2
aws ec2 describe-instance-type-offerings \
  --location-type availability-zone \
  --filters Name=instance-type,Values=g4dn.xlarge,p5.4xlarge \
  --region $REGION \
  --query "InstanceTypeOfferings[].{Type:InstanceType,AZ:Location}" --output table
```

Pick a zone (or pair of zones) where **both** types show up. `p4d.24xlarge`,
if you're going that route, has narrower regional availability than
`p5.4xlarge` — verify it the same way before relying on it.

> Same **quota ≠ capacity** rule as the other clouds: the type showing up here
> means it's offered in that AZ, not that AWS has free capacity for it right
> now. The only sure test is Section 6.

---

## 6. Smoke test: launch one GPU instance, confirm it boots, then terminate

```bash
REGION=eu-west-2
KEY_NAME=ocr-smoketest-key

aws ec2 create-key-pair --key-name $KEY_NAME --region $REGION \
  --query "KeyMaterial" --output text > $KEY_NAME.pem
chmod 400 $KEY_NAME.pem

# choose availability zone ${REGION}a / ${REGION}b / ${REGION}c
SUBNET_ID=$(aws ec2 describe-subnets \
  --region "$REGION" \
  --filters "Name=availability-zone,Values=${REGION}b" \
  --query 'Subnets[0].SubnetId' \
  --output text)

# Swap --instance-type for p4d.24xlarge to test the A100 route instead
INSTANCE_ID=$(aws ec2 run-instances \
  --region $REGION \
  --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --instance-type p5.4xlarge \
  --key-name $KEY_NAME \
  --subnet-id "$SUBNET_ID" \
  --query "Instances[0].InstanceId" --output text)

aws ec2 wait instance-running --instance-ids $INSTANCE_ID --region $REGION
aws ec2 describe-instances --instance-ids $INSTANCE_ID --region $REGION \
  --query "Reservations[0].Instances[0].State.Name" --output text
# Expected: running

# TERMINATE
aws ec2 terminate-instances --instance-ids $INSTANCE_ID --region $REGION
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID --region $REGION
aws ec2 delete-key-pair --key-name $KEY_NAME --region $REGION
rm -f $KEY_NAME.pem
```

- `running` → capacity is real; proceed.
- `InsufficientInstanceCapacity` → no free capacity right now in that
  AZ/region; try a fallback from Section 5. Not your fault.
- `VcpuLimitExceeded` → the quota request from Section 4 hasn't cleared yet —
  different problem from capacity.
- `SsmInvalidParameter: The following aliases are invalid: ...` → the AMI alias
  path doesn't exist. Canonical only publishes **`ebs-gp2`** for 22.04; the
  `ebs-gp3` variant starts at 24.04. If you'd rather run 24.04, the path is
  `/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id`.
  List what actually exists for any release with:
  ```bash
  aws ssm get-parameters-by-path --recursive --region $REGION \
    --path /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ \
    --query "Parameters[].Name" --output text
  ```

---

## 7. Capacity-proof cluster (disposable — not the real deployment)

This proves one level up from the AZ check in Section 5: that **EKS itself**
can schedule pods onto both GPU types via node groups + taints, before you
invest time in the full build. This cluster is throwaway — **delete it at the
end of this section.**

> **Real difference from AKS/GKE:** `--enable-cluster-autoscaler` (AKS) and
> `--enable-autoscaling` (GKE) are built-in and actually scale nodes.
> `eksctl`'s node group min/max only sets the underlying Auto Scaling Group's
> bounds and tags it for the Kubernetes Cluster Autoscaler — it does **not**
> install or run that autoscaler for you. Actually getting scale-to-zero on
> EKS means separately deploying Cluster Autoscaler or Karpenter, which is
> out of scope for this smoke test. To keep this test simple, each pool below
> is created with a **fixed** node instead.

```bash
REGION=eu-west-2
CLUSTER=ocr-smoketest-cluster

# Base cluster (no GPU, for the control plane + a CPU nodegroup)
eksctl create cluster \
  --name $CLUSTER --region $REGION \
  --nodegroup-name base --node-type m5.xlarge --nodes 1 \
  --managed

# T4 worker pool
eksctl create nodegroup \
  --cluster $CLUSTER --region $REGION \
  --name t4-workers --node-type g4dn.xlarge --nodes 1 \
  --node-labels "workload=layout" \
  --node-taints "nvidia.com/gpu=present:NoSchedule" \
  --managed

# H100 inference pool (swap --node-type for p4d.24xlarge for the A100 route)
eksctl create nodegroup \
  --cluster $CLUSTER --region $REGION \
  --name p5-inference --node-type p5.4xlarge --nodes 1 \
  --node-labels "workload=inference" \
  --node-taints "nvidia.com/gpu=present:NoSchedule" \
  --managed

# Connect kubectl
aws eks update-kubeconfig --name $CLUSTER --region $REGION
kubectl get nodes
```

Install the NVIDIA device plugin so pods can request `nvidia.com/gpu` (the
EKS-optimized accelerated AMI ships the drivers, but not the plugin):
```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.16.2/deployments/static/nvidia-device-plugin.yml
```

Target each workload to its pool with a `nodeSelector` (`workload: layout` /
`workload: inference`), tolerate the GPU taint, and request `nvidia.com/gpu: 1`.

Once you've confirmed `kubectl get nodes` shows both pools scheduling GPU pods
correctly, **tear the whole thing down** — this cluster has done its job:

```bash
eksctl delete cluster --name $CLUSTER --region $REGION
```

---

## 8. Cost safety (before your first GPU node boots)

- Set an **AWS Budget + alerts**: *Billing → Budgets* → 50/80/100 %.
- If you do install Cluster Autoscaler/Karpenter for real use later, keep
  `minSize: 0` on both pools so idle GPUs cost nothing.
- For interruptible batch OCR, consider **EC2 Spot** for the node groups
  (`eksctl create nodegroup --spot`) — meaningfully cheaper, but reclaimed
  with a 2-minute warning, so only for checkpoint-safe work.

---

## Appendix — Azure ↔ GCP ↔ AWS quick map

| Concept            | Azure                              | GCP                              | AWS                                    |
|---------------------|--------------------------------------|-------------------------------------|-------------------------------------------|
| Light GPU          | T4 (`NC16as_T4_v3`)                | T4 (`n1-standard-4` + accelerator) | T4 (`g4dn.xlarge`)                       |
| Heavy GPU          | A100 80GB (`NC24ads_A100_v4`)      | A100 80GB (`a2-ultragpu-1g`)     | H100 (`p5.4xlarge`) — no single-GPU A100 |
| Quota unit         | vCPUs per family                   | GPU count                        | vCPUs per family                         |
| Managed K8s        | AKS                                 | GKE                               | EKS                                       |
| Driver install     | NVIDIA device plugin (manual)      | `gpu-driver-version=default`     | Preinstalled (accelerated AMI); device plugin still manual |
| Autoscaling        | `--enable-cluster-autoscaler` (built-in) | `--enable-autoscaling` (built-in) | Manual: Cluster Autoscaler/Karpenter add-on required |
| Trial blocks GPU   | Yes → upgrade to Pay-As-You-Go     | Yes → activate full account      | No trial gate — quota is just 0 for everyone |

---

## ✅ Next step

Quota is approved, capacity is confirmed, and the disposable smoke-test
cluster from Section 7 is deleted. Head to
[`eks_deployment.md`](eks_deployment.md) to build the real 4-node-pool EKS
cluster (`gpunpt4`, `gpunph100`, `redisnp`, `apinp`) and deploy the pipeline.
