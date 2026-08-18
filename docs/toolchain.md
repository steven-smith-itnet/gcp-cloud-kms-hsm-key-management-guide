---
title: 4. Toolchain Setup
nav_order: 5
---

# Toolchain Setup
{: .no_toc }

**Phase 2 — Foundation.** Install and configure every tool this guide uses, and
lay out the repository that will hold the infrastructure code.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What you need

| Tool | Minimum version | Used for |
|:--|:--|:--|
| Google Cloud CLI | 480+ | All CLI steps |
| Terraform | 1.6+ | Primary IaC path |
| Config Connector | 1.115+ | Kubernetes-native IaC path (optional) |
| Python | 3.9+ | Client library examples, envelope encryption |
| `jq` | 1.6+ | JSON parsing in shell examples |
| `openssl` | 3.0+ | BYOK wrapping |
| Git | 2.30+ | Repo + CI |
| Checkov / Conftest | latest | Policy-as-code scanning |
| Tink (`tink-py`) | latest | Production envelope encryption |

## Install the Google Cloud CLI

**Linux (apt):**

```bash
sudo apt-get install -y apt-transport-https ca-certificates gnupg curl
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] \
https://packages.cloud.google.com/apt cloud-sdk main" \
  | sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list
sudo apt-get update && sudo apt-get install -y google-cloud-cli
gcloud version
```

**macOS:**

```bash
brew install --cask google-cloud-sdk
```

**Any platform (install script):**

```bash
curl -fsSL https://sdk.cloud.google.com | bash
exec -l "$SHELL"
```

```text
Google Cloud SDK 495.0.0
bq 2.1.9
core 2026.08.15
gcloud-crc32c 1.0.0
gsutil 5.30
```

## Authenticate and configure

```bash
# Human login for the CLI
gcloud auth login

# Separate credentials that client libraries pick up automatically
gcloud auth application-default login

gcloud config set project contoso-keymgmt-prod
gcloud config set compute/region us-east1

gcloud config configurations create keymgmt-prod 2>/dev/null || true
gcloud config configurations activate keymgmt-prod
gcloud config list
```

```text
[compute]
region = us-east1
[core]
account = steven.smith@contoso.com
project = contoso-keymgmt-prod
```

{: .tip }
> **`gcloud auth login` and `gcloud auth application-default login` are different
> credential stores.** The first is for the `gcloud` command; the second is what
> Python, Terraform, and other client libraries read via Application Default
> Credentials. Running only the first and then wondering why Terraform is
> unauthenticated is a rite of passage. Run both.

### Named configurations for switching context

```bash
for CFG in keymgmt-prod:contoso-keymgmt-prod \
           app-prod:contoso-app-prod \
           app-dev:contoso-app-dev; do
  NAME="${CFG%%:*}"; PROJECT="${CFG##*:}"
  gcloud config configurations create "$NAME" 2>/dev/null || true
  gcloud config configurations activate "$NAME"
  gcloud config set project "$PROJECT" --quiet
  gcloud config set compute/region us-east1 --quiet
done

gcloud config configurations list
gcloud config configurations activate keymgmt-prod
```

### Prove the separation of duties actually works

A five-second test that catches a misconfigured IAM binding before it matters.
Run it after you have created a key in
[Creating Keys]({% link docs/kms.md %}).

```bash
KEY_ARGS="--keyring=prod-platform --location=us-east1 --project=contoso-keymgmt-prod"

# As a member of gcp-key-admins — should SUCCEED
gcloud kms keys list $KEY_ARGS

# As the same member — should FAIL with PERMISSION_DENIED
echo -n "test" | gcloud kms encrypt --key=gcs-cmek-hsm $KEY_ARGS \
  --plaintext-file=- --ciphertext-file=/dev/null 2>&1 | tail -2
```

Expected failure:

```text
ERROR: (gcloud.kms.encrypt) PERMISSION_DENIED: Permission
'cloudkms.cryptoKeyVersions.useToEncrypt' denied on resource
'projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform/cryptoKeys/gcs-cmek-hsm'
```

{: .important }
> **If that `encrypt` succeeds, your separation of duties is broken.** The most
> common cause is `roles/owner` or `roles/editor` on the key project — both include
> Cloud KMS crypto permissions. Neither belongs on a key-management project. Check
> with:
>
> ```bash
> gcloud projects get-iam-policy contoso-keymgmt-prod \
>   --flatten="bindings[].members" \
>   --filter="bindings.role:(roles/owner OR roles/editor)" \
>   --format="table(bindings.role, bindings.members)"
> ```

## Install Terraform

```bash
wget -O- https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y terraform
terraform version
```

## Install Python and the client libraries

```bash
python3 -m venv ~/.venvs/gcpkms
source ~/.venvs/gcpkms/bin/activate

pip install --upgrade pip
pip install google-cloud-kms google-cloud-storage google-cloud-logging \
            google-auth cryptography tink

python -c "from google.cloud import kms; print('google-cloud-kms OK')"
python -c "import tink; print('tink', tink.__name__)"
```

| Package | Why |
|:--|:--|
| `google-cloud-kms` | The KMS client — key management and cryptographic operations |
| `google-auth` | Application Default Credentials |
| `cryptography` | Local AES-GCM for the hand-rolled envelope example |
| `tink` | **Production envelope encryption** — use this rather than hand-rolling |
| `google-cloud-logging` | Reading audit logs programmatically |

## Install the supporting utilities

```bash
sudo apt-get install -y jq openssl git

pip install checkov
curl -fsSL https://github.com/open-policy-agent/conftest/releases/latest/download/conftest_Linux_x86_64.tar.gz \
  | sudo tar xz -C /usr/local/bin conftest

checkov --version && conftest --version
```

## Repository layout

```text
gcp-key-management/
├── .github/
│   └── workflows/
│       ├── terraform-plan.yml        # PR: fmt, validate, checkov, plan
│       ├── terraform-apply.yml       # main: apply with Workload Identity
│       └── drift-detect.yml          # nightly: plan -detailed-exitcode
├── modules/
│   ├── kms-keyring/                  # reusable key ring module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── kms-key/
│   └── ekm-connection/
├── environments/
│   ├── keymgmt-prod/
│   │   ├── backend.tf
│   │   ├── providers.tf
│   │   ├── keyrings.tf
│   │   ├── keys.tf
│   │   ├── iam.tf
│   │   └── terraform.tfvars
│   ├── keymgmt-dev/
│   └── app-prod/                     # consumers: buckets, datasets, disks
├── config-connector/
│   ├── keyring.yaml
│   ├── keys.yaml
│   └── iam.yaml
├── policies/
│   ├── org-policies/
│   │   ├── restrict-non-cmek-services.yaml
│   │   ├── restrict-cmek-crypto-key-projects.yaml
│   │   └── allowed-protection-levels.yaml
│   ├── custom-constraints/
│   │   └── require-key-rotation.yaml
│   ├── opa/
│   │   └── kms.rego                  # Conftest rules for the Terraform plan
│   └── checkov/
│       └── KmsKeyRotationPeriod.py
├── scripts/
│   ├── envelope_encrypt.py
│   ├── tink_envelope.py
│   ├── key_inventory.py              # audit evidence export
│   ├── rotation_report.sh
│   └── byok/
│       ├── 01-create-import-job.sh
│       ├── 02-wrap-key.sh            # runs on the OFFLINE ceremony host
│       └── 03-import.sh
├── sql/
│   └── audit_queries.sql             # BigQuery forensics
├── docs/
│   └── key-inventory.md              # generated; the auditor-facing register
├── .gitignore
└── README.md
```

```bash
mkdir -p gcp-key-management/{.github/workflows,modules/{kms-keyring,kms-key,ekm-connection},environments/{keymgmt-prod,keymgmt-dev,app-prod},config-connector,policies/{org-policies,custom-constraints,opa,checkov},scripts/byok,sql,docs}
cd gcp-key-management && git init -b main
```

### The `.gitignore` that keeps you out of trouble

```bash
cat > .gitignore <<'GITIGNORE'
# Terraform
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash.log

# NEVER COMMIT — service account keys
*-key.json
*-sa.json
credentials.json
application_default_credentials.json

# NEVER COMMIT — BYOK ceremony material
*.pem
*.key
*.wrapped
*-target-key*
import-job-*.pub

# Credentials
.env
*.tfvars.secret
GITIGNORE
```

{: .warning }
> **Service account key files are the most commonly leaked Google Cloud
> credential**, and the reason
> `iam.disableServiceAccountKeyCreation` is in the baseline org policies from
> [Organization Setup]({% link docs/account-setup.md %}). If you have enforced
> that constraint, this `.gitignore` entry is a backstop for a file that should
> not exist. If you have not, enforce it — Workload Identity Federation replaces
> service account keys for CI, and attached service accounts replace them
> everywhere else.

## Terraform backend bootstrap

State for key infrastructure is itself sensitive: it contains key resource names,
IAM bindings, and rotation configuration.

```bash
gcloud config set project contoso-keymgmt-prod
PROJECT_NUMBER=$(gcloud projects describe contoso-keymgmt-prod \
  --format="value(projectNumber)")
BUCKET="gs://contoso-tfstate-keymgmt-${PROJECT_NUMBER}"
LOCATION=us-east1

# The chicken-and-egg key: created by hand, so Terraform can create the rest
gcloud kms keyrings create bootstrap --location="$LOCATION" 2>/dev/null || true

gcloud kms keys create tfstate \
  --keyring=bootstrap --location="$LOCATION" \
  --purpose=encryption --protection-level=hsm \
  --rotation-period=90d --next-rotation-time="$(date -u -d '+90 days' +%Y-%m-%dT%H:%M:%SZ)" \
  --labels=environment=prod,managed-by=bootstrap 2>/dev/null || true

KEY_NAME="projects/contoso-keymgmt-prod/locations/${LOCATION}/keyRings/bootstrap/cryptoKeys/tfstate"

# The Cloud Storage service agent must be able to use the key
GCS_SA="service-${PROJECT_NUMBER}@gs-project-accounts.iam.gserviceaccount.com"
gcloud kms keys add-iam-policy-binding tfstate \
  --keyring=bootstrap --location="$LOCATION" \
  --member="serviceAccount:${GCS_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# State bucket: versioned, uniform access, CMEK-encrypted, public access blocked
gcloud storage buckets create "$BUCKET" \
  --location="$LOCATION" \
  --uniform-bucket-level-access \
  --public-access-prevention \
  --default-encryption-key="$KEY_NAME"

gcloud storage buckets update "$BUCKET" --versioning

gcloud storage buckets describe "$BUCKET" \
  --format="yaml(name,location,default_kms_key,versioning,uniform_bucket_level_access)"
```

```yaml
default_kms_key: projects/contoso-keymgmt-prod/locations/us-east1/keyRings/bootstrap/cryptoKeys/tfstate
location: US-EAST1
name: contoso-tfstate-keymgmt-123456789012
uniform_bucket_level_access:
  enabled: true
versioning:
  enabled: true
```

{: .important }
> **The Cloud Storage service agent binding is not optional and is easy to miss.**
> Without `roles/cloudkms.cryptoKeyEncrypterDecrypter` on the key, bucket creation
> fails with a permission error that names the *service account*, not your user —
> which makes it look like a problem with your own access. Every CMEK integration
> in this guide follows the same pattern: find the service agent, grant it on the
> key. See [IAM]({% link docs/iam.md %}).

```hcl
# environments/keymgmt-prod/backend.tf
terraform {
  required_version = ">= 1.6"

  backend "gcs" {
    bucket = "contoso-tfstate-keymgmt-123456789012"
    prefix = "keymgmt-prod"
  }

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 6.0"
    }
  }
}
```

```hcl
# environments/keymgmt-prod/providers.tf
provider "google" {
  project = var.project_id
  region  = var.region

  default_labels = {
    managed-by = "terraform"
    repository = "gcp-key-management"
    environment = var.environment
  }
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}
```

Initialize:

```bash
cd environments/keymgmt-prod
terraform init
```

```text
Initializing the backend...
Successfully configured the backend "gcs"!
Initializing provider plugins...
- Installing hashicorp/google v6.8.0...
Terraform has been successfully initialized!
```

{: .warning }
> **`terraform destroy` cannot remove key rings or keys.** The Google provider
> knows this: it removes them from state and prints a warning, leaving the
> resources in place. A `terraform destroy` on a key environment therefore
> succeeds while changing almost nothing — and a subsequent `terraform apply`
> either re-imports them or fails on "already exists." Plan for this in your
> ephemeral-environment strategy: **delete the whole project**, which is the only
> real cleanup.

## Toolchain checklist

| # | Check | Command |
|:--|:--|:--|
| 1 | `gcloud` installed and current | `gcloud version` |
| 2 | Both auth flows completed | `gcloud auth list` and `gcloud auth application-default print-access-token` |
| 3 | Correct project and region set | `gcloud config list` |
| 4 | Key admins **cannot** encrypt | The SoD test above |
| 5 | No `roles/owner` or `roles/editor` on the key project | `get-iam-policy` filter |
| 6 | Terraform installed | `terraform version` |
| 7 | `google-cloud-kms` importable | `python -c "from google.cloud import kms"` |
| 8 | Repo scaffolded with `.gitignore` | `git status` |
| 9 | State bucket is CMEK-encrypted and versioned | `gcloud storage buckets describe` |
| 10 | `terraform init` succeeds | `terraform init` |

---

[Next: Creating Keys]({% link docs/kms.md %}){: .btn .btn-primary }
