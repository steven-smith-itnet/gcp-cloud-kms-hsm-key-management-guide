# Google Cloud KMS &amp; Cloud HSM — Key Management Deployment Guide

A sequential, command-level guide to planning, deploying, automating, and
governing cryptographic key management on Google Cloud — from an empty
organization through to a governed, monitored, HSM-backed production service.

**📖 Read the guide: <https://itnet-steven-smith.github.io/gcp-cloud-kms-hsm-key-management-guide/>**

---

## What this covers

| Phase | Section | Contents |
|:--|:--|:--|
| 1 — Plan | Overview, Architecture | The four protection levels, the key ring / key / version model, and the decisions that are **permanent** |
| 2 — Foundation | Organization Setup, Toolchain | Org, folders, projects, **Data Access audit logs**, baseline org policies, `gcloud`/Terraform/Config Connector/Python, CMEK-encrypted state |
| 3 — Build | Creating Keys | The same key ring and key five ways — Console, `gcloud`, Terraform, Config Connector, Python client |
| 4 — IAM | IAM &amp; Access Control | Key-scoped bindings, service agents, IAM conditions, PAM, Workload Identity Federation |
| 5 — Operate | Key Operations, Backup &amp; DR | Rotation and re-encryption, BYOK import jobs, CMEK across GCS/BigQuery/Compute/SQL/GKE, Tink envelope encryption, Autokey, version destruction |
| 6 — HSM tier | Cloud HSM &amp; External Keys | FIPS 140-2 Level 3 keys, **key attestations**, Cloud EKM and EKM via VPC |
| 7 — Observe | Logging &amp; SIEM, Monitoring | Cloud Audit Logs, BigQuery forensics, Chronicle, Splunk, canaries, log-based alerting, the on-call runbook |
| 8 — Govern | CI/CD, Policy as Code, Compliance | GitHub Actions with Workload Identity Federation, org policy and **custom constraints**, OPA/Checkov gates, and NIST/ISO 27001/SOC 2/PCI DSS mappings |
| 9 — Verify | Verification, Cost | An executable acceptance-test suite, dependency mapping, incident runbooks, and the cost model |

**29 pages.** Every step is shown as a runnable command with expected output.
Steps that cannot be automated — account and organization creation, BYOK key
ceremonies — are called out explicitly so they can be planned into a change
window.

## What is different about Google Cloud

The guide calls these out where they matter, because they change the design:

- **Key rings and keys cannot be deleted.** Only versions are destroyed. Naming
  mistakes and test keys are permanent.
- **`roles/cloudkms.admin` excludes cryptographic operations by default** —
  separation of duties is structural rather than constructed.
- **Cloud HSM is a flag, not a project.** No cluster, no ceremony, no backups —
  FIPS 140-2 Level 3 for the cost of `--protection-level=hsm`.
- **Key attestations are customer-visible**, giving cryptographic proof of
  hardware generation that the other clouds do not expose.
- **Ciphertext carries its own version**, so rotation is transparent.
- **There is no cross-region key replication** — DR must be designed explicitly.
- **Data Access audit logs are off by default**, and they are the ones that
  answer "who used this key?"

## Automation techniques demonstrated

- **Google Cloud Console** — click-paths with every field explained
- **`gcloud`** — idempotent provisioning scripts, day-two operations, incident response
- **Terraform** — reusable `kms-keyring` and `kms-key` modules, `import` blocks, and the `destroy` caveat specific to Cloud KMS
- **Config Connector** — Kubernetes-native key management with GitOps, `deletion-policy: abandon`, and Argo CD
- **Python client library** — a key provisioner with CRC32C integrity checks, an audit-evidence inventory exporter, and error handling for the failures that actually occur
- **Tink** — production envelope encryption with a bounded DEK cache
- **GitHub Actions** — Workload Identity Federation (no service account keys), plan/policy-gate/apply, nightly drift detection; plus a Cloud Build alternative
- **Policy as code** — OPA/Rego on the Terraform plan, Checkov custom checks, organization policy constraints, and **custom constraints** with CEL
- **BigQuery SQL** — key-usage forensics, decrypt-volume anomaly detection, and logging-health checks

## Companion guides

This is one of three parallel guides sharing the same structure, so the same
concept can be compared across providers:

- [AWS KMS &amp; CloudHSM](https://github.com/itnet-steven-smith/aws-kms-cloudhsm-key-management-guide)
- [Azure Key Vault &amp; Managed HSM](https://github.com/itnet-steven-smith/azure-key-vault-managed-hsm-key-management-guide)
- **Google Cloud KMS &amp; Cloud HSM** — this repository

## Building locally

```bash
bundle install
bundle exec jekyll serve
# http://127.0.0.1:4000/gcp-cloud-kms-hsm-key-management-guide/
```

The published site is built and deployed by
[`.github/workflows/pages.yml`](.github/workflows/pages.yml) on every push to
`main`.

## Disclaimer

An independent technical walkthrough based on publicly available Google Cloud
documentation. Not affiliated with, endorsed by, or reviewed by Google. Cloud
APIs, Console layouts, and pricing change frequently — verify every step against
current official documentation before use in production, and validate all cost
figures against current pricing pages.

## License

Documentation and code samples are provided as-is for educational and reference
purposes.
