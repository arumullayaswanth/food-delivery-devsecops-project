# Food Delivery — DevSecOps Project (Step-by-Step Guide)

## Domain: `tagent.cfd` | Branch: `master`

---

## Architecture

![DevSecOps Architecture](Architecture/devsecops%20project%20Architecture.jpg)

---

## What This Project Does

You push code → GitHub Actions runs **10 security stages** → deploys to **AWS EKS** → app is live at `https://tagent.cfd`

You click **destroy** → everything is deleted → your AWS bill becomes **$0**

---

## What Gets Created (Automatically by Terraform)

| # | Resource | Purpose | Cost/Day |
|---|----------|---------|----------|
| 1 | VPC | Network for everything | $0 |
| 2 | 2 Public Subnets | For bastion + load balancer | $0 |
| 3 | 2 Private Subnets | For EKS pods | $0 |
| 4 | Internet Gateway | Internet access for public subnets | $0 |
| 5 | NAT Gateway | Internet access for private subnets | ~$1.08 |
| 6 | Elastic IP | Static IP for bastion | ~$0.12 |
| 7 | EKS Cluster (Auto Mode) | Kubernetes — runs your app | ~$2.40 |
| 8 | Bastion EC2 (t3.medium) | SonarQube + kubectl (tools auto-installed) | ~$1.00 |
| 9 | 3 ECR Repositories | Stores Docker images | $0 |
| 10 | AWS Secrets Manager (4 secrets) | Stores all passwords | ~$0.05 |
| 11 | KMS Key | Encrypts EKS secrets | ~$0.03 |
| 12 | Security Groups | Firewall rules | $0 |
| 13 | IAM Policies | Permissions | $0 |
| **Total** | | | **~$5-10/day** |

---

## What Gets Deleted When You Click "Destroy"

**EVERYTHING above** → deleted completely → bill = $0

Things that stay (cost $0):
- S3 bucket (terraform state file)
- DynamoDB table (state lock)
- OIDC provider + IAM role
- ACM certificate
- Route53 hosted zone ($0.50/month)

---

## Step-by-Step Deployment

---

### PART 1: AWS Console Setup (One Time Only)

---

#### Step 1: Login to AWS Console

1. Go to https://console.aws.amazon.com
2. Login with your AWS account
3. Make sure you're in **ap-south-1 (Mumbai)** region (top-right corner)

---

#### Step 2: Create S3 Bucket (Terraform State Storage)

**Why:** Terraform saves what it created in this bucket. Without it, Terraform forgets everything.

1. Search **"S3"** in AWS Console search bar → Click it
2. Click **"Create bucket"**
3. Fill in:
   - Bucket name: `food-delivery-terraform-state-0000`
   - Region: **Asia Pacific (Mumbai) ap-south-1**
4. Scroll down → **Bucket Versioning** → Click **Enable**
5. Scroll down → **Default encryption** → Select **SSE-S3 (AES-256)**
6. Leave "Block all public access" **checked** ✅
7. Click **"Create bucket"**

✅ Done! Remember the bucket name.

---

#### Step 3: Create DynamoDB Table (State Lock)

**Why:** Prevents two people from running Terraform at the same time.

1. Search **"DynamoDB"** in AWS Console → Click it
2. Click **"Create table"**
3. Fill in:
   - Table name: `food-delivery-terraform-state-lock-0000`
   - Partition key: `LockID` (type: **String**)
4. Leave everything else as default
5. Click **"Create table"**

✅ Done!

---

#### Step 4: Create OIDC Identity Provider (GitHub Trust)

**Why:** This tells AWS "I trust GitHub. When GitHub Actions says it's from my repo, give it access." No passwords stored anywhere.

1. Search **"IAM"** in AWS Console → Click it
2. Left sidebar → Click **"Identity providers"**
3. Click **"Add provider"**
4. Fill in:
   - Provider type: **OpenID Connect**
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Click **"Get thumbprint"**
   - Audience: `sts.amazonaws.com`
5. Click **"Add provider"**

✅ Done!

---

#### Step 5: Create IAM Role for GitHub Actions

**Why:** This role has permissions to create/delete AWS resources. GitHub Actions uses this role via OIDC (no passwords needed).

1. Go to **IAM** → Left sidebar → **"Roles"** → Click **"Create role"**
2. Select **"Web identity"**
3. Fill in:
   - Identity provider: `token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
4. Click **"Next"**
5. Search and check **`AdministratorAccess`**
6. Click **"Next"**
7. Role name: `food-delivery-GitHubActions-Terraform-Role-0000`
8. Click **"Create role"**

**Now edit the trust policy:**

9. Go to **IAM → Roles** → Click on `food-delivery-GitHubActions-Terraform-Role-0000`
10. Click **"Trust relationships"** tab → Click **"Edit trust policy"**
11. Replace everything with this (change `YOUR_ACCOUNT_ID` to your 12-digit AWS account ID):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:arumullayaswanth/food-delivery-devsecops-project:*"
        }
      }
    }
  ]
}
```

12. Click **"Update policy"**
13. Go back to the role → Copy the **ARN** at the top. It looks like:
    ```
    arn:aws:iam::123456789012:role/food-delivery-GitHubActions-Terraform-Role-0000
    ```
14. **Save this ARN** — you need it in Step 8.

✅ Done!

---

#### Step 6: Create ACM Certificate (HTTPS for tagent.cfd)

**Why:** So your website has HTTPS (the lock icon in browser).

1. Search **"Certificate Manager"** in AWS Console → Click it
2. Click **"Request a certificate"**
3. Select **"Request a public certificate"** → Click **"Next"**
4. Domain names:
   - Add: `tagent.cfd`
   - Click **"Add another name to this certificate"**
   - Add: `*.tagent.cfd`
5. Validation method: **DNS validation**
6. Click **"Request"**
7. You'll see the certificate with status "Pending validation"
8. **Don't close this page** — you need it in Step 7

✅ Certificate requested! (It will be validated after Step 7)

---

#### Step 7: Create Route53 Hosted Zone (DNS for tagent.cfd)

**Why:** Route53 manages DNS records that point `tagent.cfd` to your app.

1. Search **"Route 53"** in AWS Console → Click it
2. Click **"Hosted zones"** → Click **"Create hosted zone"**
3. Fill in:
   - Domain name: `tagent.cfd`
   - Type: **Public hosted zone**
4. Click **"Create hosted zone"**
5. You'll see **4 NS records** (nameservers). They look like:
   ```
   ns-123.awsdns-45.com
   ns-678.awsdns-90.net
   ns-111.awsdns-22.org
   ns-333.awsdns-44.co.uk
   ```
6. **Go to your domain registrar** (where you bought `tagent.cfd`) → Update nameservers to these 4 values
7. Wait 5-30 minutes for DNS propagation

**Now validate the ACM certificate:**

8. Go back to **Certificate Manager** → Click on your certificate
9. Click **"Create records in Route 53"** → Click **"Create records"**
10. Wait for certificate status: **Issued** ✅

✅ Done!

---

### PART 2: GitHub Setup

---

#### Step 8: Add GitHub Variables

**Why:** The pipeline reads these values when it runs. Nothing is hardcoded in code.

1. Go to your GitHub repo: `github.com/arumullayaswanth/food-delivery-devsecops-project`
2. Click **Settings** (tab at top)
3. Left sidebar → **Secrets and variables** → Click **"Actions"**
4. Click the **"Variables"** tab
5. Click **"New repository variable"** for EACH of these:

| Variable Name | What To Put | Example |
|---|---|---|
| `AWS_REGION` | Your AWS region | `ap-south-1` |
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID | `123456789012` |
| `AWS_ROLE_ARN` | The role ARN from Step 5 | `arn:aws:iam::123456789012:role/food-delivery-GitHubActions-Terraform-Role-0000` |
| `TF_STATE_BUCKET` | S3 bucket name from Step 2 | `food-delivery-terraform-state-0000` |
| `TF_LOCK_TABLE` | DynamoDB table from Step 3 | `food-delivery-terraform-state-lock-0000` |
| `APP_URL` | Your domain | `tagent.cfd` |

**No GitHub Secrets needed. All sensitive values come from AWS Secrets Manager.**

✅ Done!

---

### PART 3: Create Infrastructure (One Click)

---

#### Step 9: Run Terraform Apply (Creates Everything)

1. Go to your GitHub repo → Click **"Actions"** tab
2. Left sidebar → Click **"EKS Terraform"** (`terraform-infra.yml`)
3. Click **"Run workflow"** (right side)
4. Select: **`apply`**
5. Click **"Run workflow"** (green button)
6. **Wait ~15-20 minutes** (EKS cluster takes time)
7. When it's green ✅ → Click on the run → Click **"Summary"** tab
8. You'll see everything that was created, cost estimate, and connection commands

**What happens automatically:**
- VPC + subnets created
- EKS cluster created (Auto Mode — no node groups to manage)
- 3 ECR repositories created
- Bastion EC2 created → **tool.sh runs automatically** (installs kubectl, Helm, Docker, SonarQube)
- Secrets Manager secrets created (with placeholder values)
- KMS key created
- All IAM policies attached
- **External Secrets Operator installed** (auto-syncs AWS Secrets Manager → Kubernetes)
- **Falco installed** (runtime security monitoring)

✅ Infrastructure is live!

---

### PART 4: Setup Bastion Server

---

#### Step 10: Connect to Bastion (SSM — No SSH Key Needed)

**Why:** You need to access the bastion to configure SonarQube and run kubectl commands.

**AWS Console (easiest):**
1. Go to **AWS Console → EC2 → Instances**
2. Select `food-delivery-bastion` → Click **Connect**
3. Click **Session Manager** tab → Click **Connect**
4. A terminal opens in your browser — no SSH key needed

**Note:** All tools (kubectl, Helm, Docker, SonarQube) are already installed automatically via user data. No manual installation needed.

---

#### Step 11: Connect to EKS from Bastion

```bash
aws eks update-kubeconfig --name food-delivery-cluster --region ap-south-1
kubectl get nodes
```

**With EKS Auto Mode**: You'll see no nodes yet. Nodes appear automatically when you deploy pods.

---

#### Step 12: Setup SonarQube

**Why:** SonarQube is already running (Docker container started automatically via user data). You just need to login and generate a token.

1. Get bastion public IP from AWS Console → EC2 → Instances → `food-delivery-bastion` → copy **Public IPv4**

2. Open in browser: `http://<BASTION_IP>:9000`

3. Login: `admin` / `admin`

4. It will ask you to change the password → change it

5. Click your **profile icon** (top-right corner) → **My Account** → **Security**

6. Under **Generate Tokens**:
   - Name: `github-actions`
   - Type: `User Token`
   - Expires in: `No expiration`
   - Click **Generate**

7. **Copy the token** (looks like `squ_abc123...`) — you won't see it again

8. Go to **AWS Console → Secrets Manager → `food-delivery/sonarqube`**
   - Click **Retrieve secret value** → Click **Edit**
   - Update the values:

   | Key | Value |
   |-----|-------|
   | `SONAR_HOST_URL` | `http://<BASTION_IP>:9000` |
   | `SONAR_TOKEN` | `squ_your_token_here` |

   - Click **Save**

9. Also update **`food-delivery/pipeline`** with the same values:
   - Go to **Secrets Manager → `food-delivery/pipeline`**
   - Click **Retrieve secret value** → Click **Edit**

   | Key | Value |
   |-----|-------|
   | `SONAR_HOST_URL` | `http://<BASTION_IP>:9000` |
   | `SONAR_TOKEN` | `squ_your_token_here` |

   - Click **Save**

✅ SonarQube ready! The pipeline fetches the token from Secrets Manager automatically.

---

#### Step 13: Put Real Secrets in AWS Secrets Manager

**Why:** Replace the placeholder `CHANGE_ME` values with your actual credentials.

---

**Step 13.1 — Generate your JWT Secret (run on bastion):**

```bash
openssl rand -base64 48
```

Copy the output — you'll paste it in the next step. Keep it secret, never share it.

---

**Step 13.2 — Get your Stripe Secret Key:**

1. Go to [dashboard.stripe.com](https://dashboard.stripe.com) → Login
2. Click **Developers** (top-right) → **API keys**
3. Under **Standard keys** → Click **Reveal test key** → Copy the key
   - Test key starts with `sk_test_` — no real money charged
   - Live key starts with `sk_live_` — real payments processed

---

**Step 13.3 — Update `food-delivery/app-secrets`:**

Go to **AWS Console → Secrets Manager → `food-delivery/app-secrets`** → **Retrieve secret value** → **Edit**

| Key | Value |
|-----|-------|
| `MONGODB_URI` | `mongodb://foodadmin:FoodSecure2024@mongodb.food-delivery.svc.cluster.local:27017/food-delivery?authSource=admin` |
| `JWT_SECRET` | paste the value from Step 13.1 |
| `STRIPE_SECRET_KEY` | paste the key from Step 13.2 |

> **Note:** `MONGODB_URI` is already pre-filled — MongoDB runs inside the cluster, no Atlas needed.

Click **Save**

---

**Step 13.4 — Update `food-delivery/database`:**

Go to **Secrets Manager → `food-delivery/database`** → **Retrieve secret value** → **Edit**

| Key | Value |
|-----|-------|
| `DB_HOST` | `mongodb.food-delivery.svc.cluster.local` |
| `DB_NAME` | `food-delivery` |
| `DB_USERNAME` | `foodadmin` |
| `DB_PASSWORD` | `FoodSecure2024` |
| `DB_PORT` | `27017` |

Click **Save**

---

**Step 13.5 — Force sync secrets to Kubernetes** (run on bastion):

```bash
kubectl annotate externalsecret food-delivery-secrets -n food-delivery force-sync=$(date +%s) --overwrite
```

✅ Secrets are now in Kubernetes!

---

### PART 5: Deploy the Application

---

#### Step 14: Run the DevSecOps Pipeline (Deploy)

1. Go to **GitHub → Actions** → Left sidebar → Click **"DevSecOps Pipeline - Food Delivery (Production)"** (`devsecops-pipeline.yml`)
2. Click **"Run workflow"**
3. Select: **`deploy`**
4. Click **"Run workflow"**

**What happens automatically (10 stages):**

```
Stage 1:  Gitleaks (secret scanning)        ✅
Stage 2:  SonarQube (code quality)          ✅
Stage 3:  Trivy (dependency scan)           ✅
Stage 4:  Docker build (distroless images)  ✅
Stage 5:  Trivy (image scan)               ✅
Stage 6:  Checkov (IaC scan)               ✅
Stage 7:  Push to ECR (OIDC auth)          ✅
Stage 8:  Deploy to EKS                    ✅
Stage 9:  OWASP ZAP (DAST)                ✅
Stage 10: Falco validation                  ✅
```

✅ App is deployed!

---

#### Step 15: Add DNS Records (Point Domain to ALB)

**Why:** Tell Route53 to send traffic from `tagent.cfd` to your load balancer.

1. Get the ALB URL (from bastion or from pipeline output):
```bash
kubectl get ingress -n food-delivery
```
Copy the ADDRESS (looks like: `k8s-fooddeli-xxx.ap-south-1.elb.amazonaws.com`)

2. Go to **AWS Console → Route 53 → tagent.cfd hosted zone**

3. Create record for frontend:
   - Record name: (leave empty — this is for `tagent.cfd`)
   - Record type: **A**
   - Toggle **"Alias"** ON
   - Route traffic to: **"Alias to Application and Classic Load Balancer"**
   - Region: **Asia Pacific (Mumbai)**
   - Select your ALB from dropdown
   - Click **"Create records"**

4. Create record for admin panel:
   - Record name: `admin`
   - Record type: **A**
   - Toggle **"Alias"** ON
   - Same ALB as above
   - Click **"Create records"**

5. Wait 2-5 minutes → Open browser:
   - `https://tagent.cfd` → Frontend ✅
   - `https://tagent.cfd/api/food` → Backend API ✅
   - `https://admin.tagent.cfd` → Admin Panel ✅

✅ **YOUR APP IS LIVE!** 🎉

---

### PART 6: Destroy Everything (Bill → $0)

---

#### Step 16: One-Click Destroy

1. Go to **GitHub → Actions** → Left sidebar → **"EKS Terraform"** (`terraform-infra.yml`)
2. Click **"Run workflow"**
3. Select: **`destroy`**
4. In "confirm_destroy" field: type **`yes`**
5. Click **"Run workflow"**
6. Wait ~10-15 minutes

**What happens automatically:**
- Uninstalls all Helm releases (Falco, External Secrets)
- Deletes all Kubernetes services/ingresses (removes ALBs)
- Deletes all PVCs (removes EBS volumes)
- Runs `terraform destroy` (removes all infrastructure)
- Cleans up orphaned LBs, EBS volumes, EIPs, NAT gateways, snapshots
- Final verification: confirms everything is deleted

✅ **Bill = $0**

---

#### Step 17: Recreate Later (When You Need It Again)

1. Go to **GitHub → Actions → EKS Terraform** (`terraform-infra.yml`)
2. Run workflow → Select **`apply`**
3. Wait 15-20 minutes
4. Everything comes back (repeat Steps 10-17)

---

## Quick Reference

### Connect to Bastion
```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=food-delivery-bastion" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text --region ap-south-1)
aws ssm start-session --target $INSTANCE_ID --region ap-south-1
```

### Connect to EKS
```bash
aws eks update-kubeconfig --name food-delivery-cluster --region ap-south-1
kubectl get pods -n food-delivery
```

### Check App Status
```bash
kubectl get pods -n food-delivery
kubectl get svc -n food-delivery
kubectl get ingress -n food-delivery
```

### View Logs
```bash
kubectl logs -f deployment/food-delivery-backend -n food-delivery
kubectl logs -f deployment/food-delivery-frontend -n food-delivery
```

### Force Secret Sync
```bash
kubectl annotate externalsecret food-delivery-secrets -n food-delivery force-sync=$(date +%s) --overwrite
```

### Falco (Runtime Security)
```bash
# Check if Falco pods are running
kubectl get pods -n falco

# Check Falco DaemonSet status
kubectl get daemonset falco -n falco

# View Falco alerts (real-time)
kubectl logs -f -l app.kubernetes.io/name=falco -n falco

# View only critical/warning alerts
kubectl logs -l app.kubernetes.io/name=falco -n falco | grep -i "Warning\|Critical\|Error"

# Check Falco Sidekick UI (forward port to access from browser)
kubectl port-forward svc/falco-falcosidekick-ui -n falco 2802:2802
# Then open: http://localhost:2802
```

---

## Security Summary

| What | How |
|------|-----|
| No AWS keys in GitHub | OIDC authentication |
| No secrets in code | AWS Secrets Manager |
| No SSH keys | SSM Session Manager |
| No root containers | Distroless images + USER 65534 |
| No open ports | Network Policies (deny-all default) |
| No manual node management | EKS Auto Mode |
| No unscanned code | SonarQube SAST + Trivy SCA |
| No vulnerable images | Trivy image scan (fail on HIGH/CRITICAL) |
| No insecure k8s config | Checkov + PSS Restricted |
| No runtime threats | Falco eBPF monitoring |
| No unpatched deps | Dependabot daily checks |
| HTTPS everywhere | ACM certificate + ALB |

---

## File Structure

```
food-delivery-devsecops-project/
├── .github/workflows/
│   ├── terraform-infra.yml        ← One-click apply/destroy
│   ├── devsecops-pipeline.yml     ← 10-stage security pipeline
│   └── dependabot.yml             ← Auto-update dependencies
├── terraform/
│   ├── provider.tf + backend.tf   ← AWS + S3 state
│   ├── vpc.tf                     ← Network
│   ├── eks.tf                     ← EKS Auto Mode
│   ├── ecr.tf                     ← Image registries
│   ├── bastion.tf                 ← Jump server + SonarQube
│   ├── secrets-manager.tf         ← All secrets
│   ├── oidc.tf                    ← GitHub → AWS trust (references only)
│   ├── outputs.tf                 ← Shows what was created
│   └── variables.tf               ← Configuration
├── k8s/
│   ├── namespace.yaml             ← PSS restricted
│   ├── *-deployment.yaml          ← Hardened pods (3 apps)
│   ├── *-service.yaml             ← ClusterIP services
│   ├── ingress.yaml               ← ALB + HTTPS (tagent.cfd)
│   ├── networkpolicy.yaml         ← Zero-trust networking
│   └── storageclass.yaml          ← EBS gp3
├── scripts/
│   ├── tool.sh                    ← Bastion setup (runs automatically)
│   ├── install-external-secrets.sh ← Run manually on bastion
│   └── uninstall-helm.sh          ← Cleanup helper
├── backend/Dockerfile             ← Distroless Node.js
├── frontend/Dockerfile            ← Distroless nginx
├── admin/Dockerfile               ← Distroless nginx
├── .gitleaks.toml                 ← Secret scanning rules
├── .pre-commit-config.yaml        ← Pre-commit hooks
├── .trivyignore                   ← Trivy exceptions
├── .zap/rules.tsv                 ← OWASP ZAP rules
├── sonar-project.properties       ← SonarQube config
├── CODEOWNERS                     ← Security team reviews
└── README-DEVSECOPS.md            ← This file
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| SSM won't connect | Wait 5 min for instance to boot. Check IAM role has `AmazonSSMManagedInstanceCore` |
| SonarQube not loading | Connect to bastion → `sudo docker logs sonarqube` → check if container is running |
| Pipeline fails at SonarQube | Check Secrets Manager has correct SONAR_TOKEN + SONAR_HOST_URL |
| EKS nodes not appearing | Normal with Auto Mode — nodes appear only when pods are scheduled |
| Destroy fails | It retries automatically. Orphaned resources cleaned in post-destroy steps |
| ALB not created | Check ingress has `ingressClassName: alb` and EKS Auto Mode has networking enabled |
| HTTPS not working | Check ACM certificate status is "Issued" and Route53 CNAME records exist |
| Secrets not syncing | Run `kubectl describe externalsecret -n food-delivery` to check errors |
