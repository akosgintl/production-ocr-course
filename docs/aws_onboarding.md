# Setting Up Your AWS Account

This is the optional AWS track. The course is built on Azure, but if you want to
replicate the pipeline on Amazon Web Services (or just prefer AWS), start here.
Once this is done, move on to `aws_gpu_prereqs.md` to get GPU access and build
the EKS cluster.

> This guide covers account creation only. It's all done in the browser — there
> are no commands until the CLI install at the end.

---

## 1. Create your AWS account

1. Go to **https://aws.amazon.com/free**.
2. Click **Create a free account**.
   > _[PLACEHOLDER: screenshot of the "Create a free account" button ➜]_
3. Enter your email and choose a name for the account (this becomes your
   **root user** — keep its credentials safe, you won't use it day-to-day).
4. Fill in your contact information.
5. Add a **credit or debit card** for verification.
   > **This does not charge you** for the verification itself. AWS uses it to
   > confirm identity. Prepaid/virtual cards are often rejected — use a normal
   > card.
   > _[PLACEHOLDER: screenshot of the payment/verification step ➜]_
6. Verify your phone number (AWS calls or texts you a code).
7. Choose a plan:
   - **Free plan** — $100 credit up front, up to $100 more for completing
     onboarding tasks, but the account **auto-closes after 6 months or when
     the credit runs out**, whichever comes first.
   - **Paid plan** — no credit, billed normally, no auto-close.
   > If you plan to actually run paid GPU instances during this course, prefer
   > **Paid plan** — an account that auto-closes mid-course because a Free-plan
   > credit ran out is a bad surprise. See the warning below either way.
8. Pick the **Basic support plan** (free) unless you already know you want more.

> **What you get:** up to $200 in credit (Free plan) plus ~30 always-free
> services for 12 months. **None of this covers GPU instances** — see below.

---

## 2. Important things to know before you go further

**GPU quota is 0 for every new account, Free or Paid.** Unlike Azure/GCP, AWS
doesn't gate GPUs behind a "trial vs. paid" switch — there's no upgrade step.
Instead, every account starts with a **0 vCPU quota** on the GPU instance
families (`G`/`VT` and `P`), full stop, and you must request an increase
through **Service Quotas** regardless of which plan you picked. Covered in
`aws_gpu_prereqs.md`.

**Brand-new accounts can still be throttled.** Even with the quota request
filed correctly, AWS's automated risk screening can delay or deny larger
increases (especially on the `P` family) for accounts with no billing history.
If that happens: generate a little ordinary paid usage first and retry, or
open a support case explaining the use case.

**AWS has no single-GPU A100 instance.** This is a real architectural
difference from Azure/GCP, not just naming — Azure's `NC24ads_A100_v4` and
GCP's `a2-highgpu-1g` both give you **one** A100 per VM; AWS's A100 instance
(`p4d.24xlarge`) only ships as a fixed **8-GPU** node, there is no smaller
size. The practical fix — a single-GPU **H100** instance (`p5.4xlarge`), which
preserves the "one GPU per node" autoscaling shape the other clouds use — is
covered in detail in `aws_gpu_prereqs.md`.

**Use IAM, not root, for everything after signup.** The root user you just
created is for account-level tasks only (billing, closing the account). Create
an IAM identity for the CLI and daily work — step 3 below.

**Quota and capacity are both per-region.** Same as Azure/GCP: pick one AWS
region and request quota there. Details on choosing one in
`aws_gpu_prereqs.md`.

---

## 3. Create an IAM user for CLI access

Don't generate access keys for the root user. Create a separate IAM identity:

1. Go to **https://console.aws.amazon.com/iam**.
2. Left menu → **Users** → **Create user**.
   > _[PLACEHOLDER: screenshot of the Create user form ➜]_
3. Name it (e.g. `ocr-course-admin`), enable **console access** if you want a
   browser login too.
4. Attach the **`AdministratorAccess`** managed policy (fine for a personal
   course account; scope this down for anything shared or long-lived).
5. Finish, then open the new user → **Security credentials** → **Create access
   key** → choose **Command Line Interface (CLI)** → create it and save the
   Access Key ID + Secret Access Key somewhere safe (shown once).
6. (Recommended) Enable **MFA** on both the root user and this IAM user.

---

## 4. Install the AWS CLI

**macOS (Homebrew)**
```bash
brew install awscli
```

**Linux**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Windows**: download the installer from
**https://awscli.amazonaws.com/AWSCLIV2.msi**

**Configure and verify**
```bash
aws configure
# paste the Access Key ID / Secret Access Key from step 3, pick a default region
aws sts get-caller-identity     # confirms who the CLI is authenticated as
```

---

## ✅ Next step

Your account, IAM user, and CLI all work. Now head to
**`aws_gpu_prereqs.md`** to request GPU quota, understand the A100 instance
size caveat, verify availability by region, and prove capacity with a
disposable EKS smoke-test cluster.
