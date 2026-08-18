---
title: 5.3 Terraform
parent: 5. Creating Keys
nav_order: 3
---

# Creating Keys with Terraform
{: .no_toc }

The production path. A reviewed, versioned, reusable module — with a large
caveat about `destroy` that is specific to Cloud KMS.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The `destroy` caveat, first

```hcl
resource "google_kms_key_ring" "example" {
  name     = "prod-platform"
  location = "us-east1"
}
```

```bash
terraform destroy
```

```text
Warning: Cannot destroy KMS Key Ring

  with google_kms_key_ring.example,

KMS Key Rings cannot be deleted from Google Cloud Platform. Terraform will
remove this key ring from its state but will not delete the resource from the
project.

Destroy complete! Resources: 1 destroyed.
```

{: .warning }
> **"Destroy complete" is not true, and neither is "1 destroyed."** The key ring
> still exists in the project, permanently. The same applies to
> `google_kms_crypto_key`. Three consequences you must design around:
>
> 1. **Ephemeral environments cannot be cleaned up by Terraform.** Delete the
>    whole project instead.
> 2. **A re-`apply` after a `destroy` fails with `alreadyExists`**, unless you
>    import the resources back into state first.
> 3. **CI that tears down and rebuilds test infrastructure will accumulate key
>    rings forever.** Use a per-run project, or a fixed set of reusable test keys.

## The reusable modules

### `modules/kms-keyring/main.tf`

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    google = { source = "hashicorp/google", version = "~> 6.0" }
  }
}

resource "google_kms_key_ring" "this" {
  project  = var.project_id
  name     = var.name
  location = var.location

  lifecycle {
    # A key ring cannot be deleted anyway, but this stops a plan from
    # pretending it can — and from silently dropping it out of state.
    prevent_destroy = true
  }
}

# Key-ring-level IAM: the right granularity for a trust domain
resource "google_kms_key_ring_iam_member" "bindings" {
  for_each = var.iam_bindings

  key_ring_id = google_kms_key_ring.this.id
  role        = each.value.role
  member      = each.value.member

  dynamic "condition" {
    for_each = each.value.condition == null ? [] : [each.value.condition]
    content {
      title       = condition.value.title
      description = condition.value.description
      expression  = condition.value.expression
    }
  }
}
```

### `modules/kms-keyring/variables.tf`

```hcl
variable "project_id" {
  type = string
}

variable "name" {
  description = "Key ring name. PERMANENT — cannot be renamed, moved, or deleted."
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,61}[a-z0-9]$", var.name))
    error_message = "Key ring names must be lowercase alphanumeric with hyphens, 3-63 chars."
  }
}

variable "location" {
  description = "PERMANENT. Must match the location of the resources this ring will encrypt."
  type        = string
}

variable "iam_bindings" {
  description = "Map of binding key -> { role, member, condition }"
  type = map(object({
    role   = string
    member = string
    condition = optional(object({
      title       = string
      description = string
      expression  = string
    }))
  }))
  default = {}
}
```

### `modules/kms-key/main.tf`

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    google = { source = "hashicorp/google", version = "~> 6.0" }
  }
}

locals {
  # Rotation is only valid for symmetric ENCRYPT_DECRYPT keys.
  rotation_supported = var.purpose == "ENCRYPT_DECRYPT"
  rotation_effective = var.rotation_period != null && local.rotation_supported
}

resource "google_kms_crypto_key" "this" {
  name     = var.name
  key_ring = var.key_ring_id
  purpose  = var.purpose

  rotation_period = local.rotation_effective ? var.rotation_period : null

  # The window in which a mistaken version destruction can be reversed.
  destroy_scheduled_duration = var.destroy_scheduled_duration

  # Only ever accept imported key versions (BYOK-only keys)
  import_only = var.import_only

  # EXTERNAL keys have no initial version — you attach one explicitly
  skip_initial_version_creation = var.skip_initial_version_creation

  version_template {
    algorithm        = var.algorithm
    protection_level = var.protection_level
  }

  labels = merge(var.labels, {
    managed-by = "terraform"
    purpose    = lower(replace(var.purpose, "_", "-"))
  })

  lifecycle {
    # A key protects real data and cannot be deleted from GCP anyway.
    # This stops Terraform from silently dropping it out of state.
    prevent_destroy = true
  }
}

resource "google_kms_crypto_key_iam_member" "bindings" {
  for_each = var.iam_bindings

  crypto_key_id = google_kms_crypto_key.this.id
  role          = each.value.role
  member        = each.value.member

  dynamic "condition" {
    for_each = each.value.condition == null ? [] : [each.value.condition]
    content {
      title       = condition.value.title
      description = condition.value.description
      expression  = condition.value.expression
    }
  }
}
```

### `modules/kms-key/variables.tf`

```hcl
variable "name" {
  description = "Key name. PERMANENT — cannot be renamed or deleted."
  type        = string
}

variable "key_ring_id" {
  type = string
}

variable "purpose" {
  description = "IMMUTABLE. ENCRYPT_DECRYPT, ASYMMETRIC_SIGN, ASYMMETRIC_DECRYPT, MAC, or RAW_ENCRYPT_DECRYPT."
  type        = string
  default     = "ENCRYPT_DECRYPT"

  validation {
    condition = contains([
      "ENCRYPT_DECRYPT", "ASYMMETRIC_SIGN", "ASYMMETRIC_DECRYPT",
      "MAC", "RAW_ENCRYPT_DECRYPT",
    ], var.purpose)
    error_message = "Unsupported purpose."
  }
}

variable "algorithm" {
  type    = string
  default = "GOOGLE_SYMMETRIC_ENCRYPTION"
}

variable "protection_level" {
  description = "IMMUTABLE. SOFTWARE, HSM, EXTERNAL, or EXTERNAL_VPC."
  type        = string
  default     = "HSM"

  validation {
    condition     = contains(["SOFTWARE", "HSM", "EXTERNAL", "EXTERNAL_VPC"], var.protection_level)
    error_message = "protection_level must be SOFTWARE, HSM, EXTERNAL, or EXTERNAL_VPC."
  }
}

variable "rotation_period" {
  description = "e.g. \"7776000s\" (90 days). Ignored for non-symmetric keys. Minimum 24h."
  type        = string
  default     = "7776000s"
}

variable "destroy_scheduled_duration" {
  description = "Window to cancel a version destruction. The API default is 24h — far too short."
  type        = string
  default     = "2592000s" # 30 days

  validation {
    condition     = can(regex("^[0-9]+s$", var.destroy_scheduled_duration))
    error_message = "Express the duration in seconds, e.g. \"2592000s\"."
  }
}

variable "import_only" {
  type    = bool
  default = false
}

variable "skip_initial_version_creation" {
  type    = bool
  default = false
}

variable "iam_bindings" {
  type = map(object({
    role   = string
    member = string
    condition = optional(object({
      title       = string
      description = string
      expression  = string
    }))
  }))
  default = {}
}

variable "labels" {
  type    = map(string)
  default = {}
}
```

### `modules/kms-key/outputs.tf`

```hcl
output "id" {
  description = "Fully qualified key ID — use this for CMEK references."
  value       = google_kms_crypto_key.this.id
}

output "name" {
  value = google_kms_crypto_key.this.name
}

output "primary_version" {
  description = "The current primary version, if the key has one."
  value       = try(google_kms_crypto_key.this.primary[0].name, null)
}
```

{: .important }
> **`destroy_scheduled_duration` defaults to 24 hours in the API and this module
> overrides it to 30 days.** That value is the entire window in which a mistaken
> `versions destroy` can be reversed. Twenty-four hours means a Friday-evening
> mistake is unrecoverable by Monday; thirty days gives a person a realistic
> chance. Set it at creation — it cannot be shortened retroactively for existing
> versions.

## Using the modules

`environments/keymgmt-prod/keyrings.tf`

```hcl
locals {
  workload_project = "contoso-app-prod"

  common_labels = {
    environment = "prod"
    owner       = "platform-team"
    cost-center = "cc-4417"
    repository  = "gcp-key-management"
  }
}

data "google_project" "workload" {
  project_id = local.workload_project
}

module "platform_ring" {
  source = "../../modules/kms-keyring"

  project_id = var.project_id
  name       = "prod-platform"
  location   = var.region

  iam_bindings = {
    admins = {
      role   = "roles/cloudkms.admin"
      member = "group:gcp-key-admins@contoso.com"
    }
    auditors = {
      role   = "roles/cloudkms.viewer"
      member = "group:gcp-key-auditors@contoso.com"
    }
  }
}

module "payments_ring" {
  source = "../../modules/kms-keyring"

  project_id = var.project_id
  name       = "prod-payments"
  location   = var.region

  # Deliberately a NARROWER set of principals than the platform ring
  iam_bindings = {
    admins = {
      role   = "roles/cloudkms.admin"
      member = "group:gcp-payments-key-admins@contoso.com"
    }
  }
}
```

`environments/keymgmt-prod/keys.tf`

```hcl
locals {
  # The service agents that will use each CMEK. These are well-known,
  # per-project identities — see the IAM section.
  project_number = data.google_project.workload.number

  service_agents = {
    gcs     = "serviceAccount:service-${local.project_number}@gs-project-accounts.iam.gserviceaccount.com"
    bq      = "serviceAccount:bq-${local.project_number}@bigquery-encryption.iam.gserviceaccount.com"
    compute = "serviceAccount:service-${local.project_number}@compute-system.iam.gserviceaccount.com"
    sql     = "serviceAccount:service-${local.project_number}@gcp-sa-cloud-sql.iam.gserviceaccount.com"
    secrets = "serviceAccount:service-${local.project_number}@gcp-sa-secretmanager.iam.gserviceaccount.com"
  }

  cmek_keys = {
    gcs-cmek-hsm     = { agent = "gcs",     data_class = "confidential" }
    bq-cmek-hsm      = { agent = "bq",      data_class = "confidential" }
    disk-cmek-hsm    = { agent = "compute", data_class = "confidential" }
    sql-cmek-hsm     = { agent = "sql",     data_class = "confidential" }
    secrets-cmek-hsm = { agent = "secrets", data_class = "confidential" }
  }
}

module "cmek" {
  source   = "../../modules/kms-key"
  for_each = local.cmek_keys

  name        = each.key
  key_ring_id = module.platform_ring.id

  purpose          = "ENCRYPT_DECRYPT"
  algorithm        = "GOOGLE_SYMMETRIC_ENCRYPTION"
  protection_level = "HSM"

  rotation_period            = "7776000s"  # 90 days
  destroy_scheduled_duration = "2592000s"  # 30 days

  # Scope each service agent to ONLY its own key
  iam_bindings = {
    service_agent = {
      role   = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
      member = local.service_agents[each.value.agent]
    }
  }

  labels = merge(local.common_labels, { data-class = each.value.data_class })
}

module "pci_chd" {
  source = "../../modules/kms-key"

  name        = "pci-chd-hsm"
  key_ring_id = module.payments_ring.id

  purpose          = "ENCRYPT_DECRYPT"
  protection_level = "HSM"
  rotation_period  = "7776000s"

  iam_bindings = {
    payments_app = {
      role   = "roles/cloudkms.cryptoKeyEncrypterDecrypter"
      member = "serviceAccount:payments-api@contoso-app-prod.iam.gserviceaccount.com"
    }
  }

  labels = merge(local.common_labels, {
    data-class = "restricted"
    compliance = "pci"
  })
}

module "artifact_signing" {
  source = "../../modules/kms-key"

  name        = "artifact-signing-hsm"
  key_ring_id = module.signing_ring.id

  purpose          = "ASYMMETRIC_SIGN"
  algorithm        = "EC_SIGN_P384_SHA384"
  protection_level = "HSM"
  rotation_period  = null   # not supported for asymmetric keys

  iam_bindings = {
    build_pipeline = {
      role   = "roles/cloudkms.signer"
      member = "serviceAccount:gha-build@contoso-app-prod.iam.gserviceaccount.com"
    }
    public_key_readers = {
      role   = "roles/cloudkms.publicKeyViewer"
      member = "group:gcp-engineering@contoso.com"
    }
  }

  labels = merge(local.common_labels, { data-class = "internal" })
}
```

`environments/keymgmt-prod/outputs.tf`

```hcl
output "key_ids" {
  description = "Fully qualified key IDs for CMEK wiring."
  value       = { for k, m in module.cmek : k => m.id }
}

# Publish to Secret Manager so workload stacks need no remote state access
resource "google_secret_manager_secret" "key_ids" {
  for_each = module.cmek

  secret_id = "kms-key-id-${each.key}"
  project   = var.project_id

  replication {
    auto {}
  }

  labels = local.common_labels
}

resource "google_secret_manager_secret_version" "key_ids" {
  for_each = module.cmek

  secret      = google_secret_manager_secret.key_ids[each.key].id
  secret_data = each.value.id
}
```

## Plan and apply

```bash
cd environments/keymgmt-prod

terraform fmt -recursive -check
terraform validate
terraform plan -out=keys.tfplan
```

```text
Terraform will perform the following actions:

  # module.platform_ring.google_kms_key_ring.this will be created
  + resource "google_kms_key_ring" "this" {
      + id       = (known after apply)
      + location = "us-east1"
      + name     = "prod-platform"
      + project  = "contoso-keymgmt-prod"
    }

  # module.cmek["gcs-cmek-hsm"].google_kms_crypto_key.this will be created
  + resource "google_kms_crypto_key" "this" {
      + destroy_scheduled_duration = "2592000s"
      + name                       = "gcs-cmek-hsm"
      + purpose                    = "ENCRYPT_DECRYPT"
      + rotation_period            = "7776000s"
      + version_template {
          + algorithm        = "GOOGLE_SYMMETRIC_ENCRYPTION"
          + protection_level = "HSM"
        }
    }

Plan: 18 to add, 0 to change, 0 to destroy.
```

```bash
terraform apply keys.tfplan
```

{: .tip }
> **Read the `protection_level` line in every plan before applying.** It is
> immutable, and a key created as `SOFTWARE` by accident is a permanent resource
> you must work around forever. It is the single most valuable thing to eyeball in
> a KMS plan.

## Importing existing resources

Terraform 1.5+ `import` blocks are reviewable in a PR — prefer them over
`terraform import`.

```hcl
# environments/keymgmt-prod/imports.tf  (delete after the first apply)
import {
  to = module.platform_ring.google_kms_key_ring.this
  id = "projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform"
}

import {
  to = module.cmek["gcs-cmek-hsm"].google_kms_crypto_key.this
  id = "projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform/cryptoKeys/gcs-cmek-hsm"
}
```

```bash
terraform plan -generate-config-out=generated.tf
terraform plan     # should converge on "0 to add, 0 to change, 0 to destroy"
```

{: .note }
> **Import is the normal recovery path after an accidental `terraform destroy`.**
> Because the resources still exist, importing them back into state is a complete
> fix — which is a small mercy given how alarming the `destroy` output looks.

## Handling a change to an immutable field

```hcl
module "cmek" {
  # ...
  protection_level = "SOFTWARE"   # changed from HSM
}
```

```text
Error: Instance cannot be destroyed

  on ../../modules/kms-key/main.tf line 14:
  14: resource "google_kms_crypto_key" "this" {

Resource module.cmek["gcs-cmek-hsm"].google_kms_crypto_key.this has
lifecycle.prevent_destroy set, but the plan calls for this resource to be
destroyed.
```

{: .important }
> **`prevent_destroy` turning a silent, destructive change into a hard error is
> exactly the behavior you want.** Without it, Terraform would plan a
> replace — destroying the key from state, creating a new one with the same name
> (which fails, because it already exists), and leaving the environment broken.
> With it, the plan stops and tells you that you changed something immutable.

## Common Terraform pitfalls with Cloud KMS

| Symptom | Cause | Fix |
|:--|:--|:--|
| `destroy` reports success but nothing is gone | Key rings and keys cannot be deleted | Delete the project instead |
| `alreadyExists` on `apply` after a `destroy` | Resource still exists, gone from state | `import` it back |
| `INVALID_ARGUMENT` on `rotation_period` | Set on a non-symmetric key | Use the `rotation_supported` local |
| Plan wants to replace a key | An immutable field changed | Read the diff; `prevent_destroy` will block it |
| Service agent permission errors on CMEK | Agent does not exist until the API is used once | `google_project_service_identity` to force creation |
| IAM binding drift | Something else is managing the same policy | Use `_iam_member`, never `_iam_policy` |

```hcl
# Force the service agent into existence before granting it
resource "google_project_service_identity" "gcs" {
  provider = google-beta
  project  = local.workload_project
  service  = "storage.googleapis.com"
}
```

{: .warning }
> **Never use `google_kms_crypto_key_iam_policy`.** The `_iam_policy` resources
> are authoritative — they overwrite the entire policy on the key, silently
> removing bindings created by anything else, including the service agents Google
> adds automatically. Use `_iam_member` (one binding) or `_iam_binding` (one role)
> instead. This is a general Google-provider rule and it bites hardest on KMS,
> where the removed binding breaks a production integration.

---

[Next: 5.4 Config Connector]({% link docs/kms-config-connector.md %}){: .btn .btn-primary }
