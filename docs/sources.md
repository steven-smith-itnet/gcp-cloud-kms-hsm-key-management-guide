---
title: Sources & References
nav_order: 18
---

# Sources &amp; References
{: .no_toc }

Official Google Cloud documentation for every service and claim in this guide.
Verify against these before running anything in production — cloud APIs and
pricing change frequently, and this guide is a walkthrough, not an authority.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Cloud KMS

| Topic | Link |
|:--|:--|
| Cloud KMS documentation | [cloud.google.com/kms/docs](https://cloud.google.com/kms/docs) |
| Key resource hierarchy | [Cloud KMS resources](https://cloud.google.com/kms/docs/resource-hierarchy) |
| Key purposes and algorithms | [Key purposes and algorithms](https://cloud.google.com/kms/docs/algorithms) |
| Protection levels | [Protection levels](https://cloud.google.com/kms/docs/reference/rest/v1/ProtectionLevel) |
| Locations | [Cloud KMS locations](https://cloud.google.com/kms/docs/locations) |
| Key rotation | [Key rotation](https://cloud.google.com/kms/docs/rotate-key) |
| Key states | [Key version states](https://cloud.google.com/kms/docs/key-states) |
| Destroying and restoring versions | [Destroy and restore key versions](https://cloud.google.com/kms/docs/destroy-restore) |
| Envelope encryption | [Envelope encryption](https://cloud.google.com/kms/docs/envelope-encryption) |
| Additional authenticated data | [Additional authenticated data](https://cloud.google.com/kms/docs/additional-authenticated-data) |
| Importing keys | [Importing a key](https://cloud.google.com/kms/docs/importing-a-key) |
| Import job | [Create an import job](https://cloud.google.com/kms/docs/create-import-job) |
| API reference | [Cloud KMS API](https://cloud.google.com/kms/docs/reference/rest) |
| Quotas | [Cloud KMS quotas](https://cloud.google.com/kms/quotas) |
| Pricing | [cloud.google.com/kms/pricing](https://cloud.google.com/kms/pricing) |

## Cloud HSM and Cloud EKM

| Topic | Link |
|:--|:--|
| Cloud HSM overview | [Cloud HSM](https://cloud.google.com/kms/docs/hsm) |
| **Key attestations** | [Verify attestations](https://cloud.google.com/kms/docs/attest-key) |
| Attestation format | [Attestation contents](https://cloud.google.com/kms/docs/attest-key#attestation_contents) |
| Cloud EKM overview | [Cloud External Key Manager](https://cloud.google.com/kms/docs/ekm) |
| Cloud EKM via VPC | [Manage EKM connections](https://cloud.google.com/kms/docs/ekm-manage-connections) |
| Coordinated external key management | [Coordinated EKM](https://cloud.google.com/kms/docs/ekm-coordinated) |
| EKM partners | [Cloud EKM partners](https://cloud.google.com/kms/docs/ekm#partners) |

## Autokey and CMEK

| Topic | Link |
|:--|:--|
| Cloud KMS Autokey | [Autokey overview](https://cloud.google.com/kms/docs/autokey-overview) |
| Enable Autokey | [Enable Autokey](https://cloud.google.com/kms/docs/enable-autokey) |
| CMEK overview | [Customer-managed encryption keys](https://cloud.google.com/kms/docs/cmek) |
| CMEK-integrated services | [CMEK-integrated services](https://cloud.google.com/kms/docs/cmek-integrations) |
| Cloud Storage CMEK | [Customer-managed encryption keys](https://cloud.google.com/storage/docs/encryption/customer-managed-keys) |
| BigQuery CMEK | [Customer-managed Cloud KMS keys](https://cloud.google.com/bigquery/docs/customer-managed-encryption) |
| Compute Engine CMEK | [Protect resources with Cloud KMS keys](https://cloud.google.com/compute/docs/disks/customer-managed-encryption) |
| Cloud SQL CMEK | [Use customer-managed encryption keys](https://cloud.google.com/sql/docs/postgres/configure-cmek) |
| GKE etcd encryption | [Encrypt secrets at the application layer](https://cloud.google.com/kubernetes-engine/docs/how-to/encrypting-secrets) |
| Secret Manager CMEK | [Customer-managed encryption keys](https://cloud.google.com/secret-manager/docs/cmek) |

## IAM and governance

| Topic | Link |
|:--|:--|
| Cloud KMS IAM roles | [Permissions and roles](https://cloud.google.com/kms/docs/reference/permissions-and-roles) |
| Separation of duties | [Separation of duties](https://cloud.google.com/kms/docs/separation-of-duties) |
| IAM conditions | [Overview of IAM Conditions](https://cloud.google.com/iam/docs/conditions-overview) |
| Policy Analyzer | [Analyze IAM policies](https://cloud.google.com/policy-intelligence/docs/analyze-iam-policies) |
| Privileged Access Manager | [PAM overview](https://cloud.google.com/iam/docs/pam-overview) |
| Workload Identity Federation | [Workload identity federation](https://cloud.google.com/iam/docs/workload-identity-federation) |
| GKE Workload Identity | [Use Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) |
| Organization policy constraints | [Organization policy constraints](https://cloud.google.com/resource-manager/docs/organization-policy/org-policy-constraints) |
| Custom constraints | [Create custom constraints](https://cloud.google.com/resource-manager/docs/organization-policy/creating-managing-custom-constraints) |
| Project liens | [Prevent project deletion](https://cloud.google.com/resource-manager/docs/project-liens) |
| Cloud Asset Inventory | [Asset inventory overview](https://cloud.google.com/asset-inventory/docs/overview) |
| Security Command Center | [SCC overview](https://cloud.google.com/security-command-center/docs/concepts-security-command-center-overview) |
| Security Health Analytics findings | [Vulnerability findings](https://cloud.google.com/security-command-center/docs/concepts-vulnerabilities-findings) |
| Cloud Audit Logs | [Cloud Audit Logs overview](https://cloud.google.com/logging/docs/audit) |
| Configuring Data Access logs | [Configure Data Access logs](https://cloud.google.com/logging/docs/audit/configure-data-access) |
| Log sinks | [Route logs to supported destinations](https://cloud.google.com/logging/docs/export/configure_export_v2) |
| Google Cloud Architecture Framework | [Landing zone design](https://cloud.google.com/architecture/landing-zones) |

## Automation and SDKs

| Topic | Link |
|:--|:--|
| `gcloud kms` reference | [gcloud kms](https://cloud.google.com/sdk/gcloud/reference/kms) |
| Terraform `google_kms_crypto_key` | [registry.terraform.io](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/kms_crypto_key) |
| Terraform `google_kms_key_ring` | [registry.terraform.io](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/kms_key_ring) |
| Config Connector KMS resources | [KMSCryptoKey](https://cloud.google.com/config-connector/docs/reference/resource-docs/kms/kmscryptokey) |
| Python client library | [google-cloud-kms](https://cloud.google.com/python/docs/reference/cloudkms/latest) |
| Cloud KMS code samples | [Samples](https://cloud.google.com/kms/docs/samples) |
| **Tink** | [Tink cryptographic library](https://developers.google.com/tink) |
| Tink + Cloud KMS | [Envelope encryption with Tink](https://developers.google.com/tink/client-side-encryption) |
| `google-github-actions/auth` | [github.com/google-github-actions/auth](https://github.com/google-github-actions/auth) |
| GitHub OIDC with Google Cloud | [Configuring OpenID Connect in Google Cloud](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform) |
| Open Policy Agent / Rego | [openpolicyagent.org](https://www.openpolicyagent.org/docs/latest/policy-language/) |
| Conftest | [conftest.dev](https://www.conftest.dev/) |
| Checkov | [checkov.io](https://www.checkov.io/) |

## Standards and compliance

| Standard | Link |
|:--|:--|
| NIST SP 800-53 Rev. 5 | [csrc.nist.gov/pubs/sp/800/53/r5/upd1/final](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final) |
| NIST SP 800-57 Part 1 — Key Management | [csrc.nist.gov/pubs/sp/800/57/pt1/r5/final](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) |
| FIPS 140-2 | [csrc.nist.gov/pubs/fips/140-2/upd2/final](https://csrc.nist.gov/pubs/fips/140-2/upd2/final) |
| FIPS 140-3 | [csrc.nist.gov/pubs/fips/140-3/final](https://csrc.nist.gov/pubs/fips/140-3/final) |
| CMVP validated modules search | [csrc.nist.gov/projects/cryptographic-module-validation-program/validated-modules](https://csrc.nist.gov/projects/cryptographic-module-validation-program/validated-modules) |
| ISO/IEC 27001:2022 | [iso.org/standard/27001](https://www.iso.org/standard/27001) |
| PCI DSS v4.0 | [pcisecuritystandards.org](https://www.pcisecuritystandards.org/document_library/) |
| SOC 2 Trust Services Criteria | [aicpa-cima.com](https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022) |
| CIS Google Cloud Foundations Benchmark | [cisecurity.org/benchmark/google_cloud_computing_platform](https://www.cisecurity.org/benchmark/google_cloud_computing_platform) |
| Google Cloud compliance offerings | [cloud.google.com/security/compliance](https://cloud.google.com/security/compliance) |
| Google Cloud shared responsibility | [Shared responsibility and shared fate](https://cloud.google.com/architecture/framework/security/shared-responsibility-shared-fate) |
| Google Cloud encryption at rest whitepaper | [Encryption at rest](https://cloud.google.com/docs/security/encryption/default-encryption) |

## Companion guides

| Cloud | Guide |
|:--|:--|
| AWS | [AWS KMS &amp; CloudHSM Key Management Guide](https://steven-smith-itnet.github.io/aws-kms-cloudhsm-key-management-guide/) |
| Azure | [Azure Key Vault &amp; Managed HSM Key Management Guide](https://steven-smith-itnet.github.io/azure-key-vault-managed-hsm-key-management-guide/) |

---

## About this guide

Prepared by [steven-smith-itnet](https://github.com/steven-smith-itnet) as an independent
technical walkthrough. It is not affiliated with, endorsed by, or reviewed by
Google.

Commands, API shapes, Console navigation, and pricing reflect publicly documented
behavior at the time of writing. **Verify everything against current official
documentation before use in production.**

[View this guide on GitHub](https://github.com/steven-smith-itnet/gcp-cloud-kms-hsm-key-management-guide){: .btn }
