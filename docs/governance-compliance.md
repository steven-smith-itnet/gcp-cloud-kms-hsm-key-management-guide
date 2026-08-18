---
title: 14. Governance & Compliance
nav_order: 15
---

# Governance &amp; Compliance Mapping
{: .no_toc }

**Phase 8 — Govern.** The FIPS posture, the control mappings, and the evidence
that turns a working system into a passed audit.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## FIPS 140 posture

| Protection level | Key material | FIPS validation |
|:--|:--|:--|
| `SOFTWARE` | Google-managed software boundary (BoringCrypto) | 140-2 Level 1 |
| `HSM` | Google-operated HSMs | **140-2 Level 3** |
| `EXTERNAL` / `EXTERNAL_VPC` | Your external key manager | Whatever your EKM holds |

{: .important }
> **Validation levels and certificate numbers change.** Before writing a FIPS claim
> into a SOC 2 description, a FedRAMP SSP, or a customer questionnaire, verify the
> current certificate on the
> [NIST CMVP validated modules search](https://csrc.nist.gov/projects/cryptographic-module-validation-program/validated-modules)
> and cite the certificate number, not a marketing page.

{: .tip }
> **Cloud HSM's attestations are the strongest FIPS evidence available on any of
> the three major clouds.** A signed, manufacturer-chained attestation proving a
> specific key version was generated inside genuine HSM hardware is something
> neither AWS KMS nor Azure Key Vault Premium exposes to customers. Capture them for
> every key carrying a regulatory claim — see
> [Cloud HSM]({% link docs/cloud-hsm.md %}#key-attestations).

```bash
# Prove every production key is HSM-backed
for RING in $(gcloud kms keyrings list --location=us-east1 \
                --project=contoso-keymgmt-prod --format="value(name.basename())"); do
  gcloud kms keys list --keyring="$RING" --location=us-east1 \
    --project=contoso-keymgmt-prod \
    --format="table[no-heading](name.basename(), versionTemplate.protectionLevel)" \
    | sed "s|^|  $RING/|"
done
```

## Control mappings

### NIST SP 800-53 Rev. 5

| Control | Requirement | How this build satisfies it | Evidence |
|:--|:--|:--|:--|
| **SC-12** | Cryptographic key establishment and management | Keys generated in FIPS 140-2 L3 HSMs; documented hierarchy | [Architecture]({% link docs/architecture.md %}); `key_inventory.py` |
| **SC-12(1)** | Availability in the event of key loss | Destroy-scheduled window, project lien, config backup | [Backup &amp; DR]({% link docs/backup-dr.md %}) |
| **SC-12(2)** | Symmetric keys from a validated module | `GOOGLE_SYMMETRIC_ENCRYPTION` at `HSM` protection | Key attestation |
| **SC-12(3)** | Asymmetric keys from an approved source | `EC_SIGN_P384_SHA384` at `HSM` | Key attestation |
| **SC-13** | FIPS-validated cryptography | AES-256-GCM, RSA-3072+, P-384 in Cloud HSM | CMVP certificate; attestation |
| **SC-28** | Protection of information at rest | CMEK on GCS, BigQuery, Compute, SQL, GKE, Pub/Sub | [CMEK Integrations]({% link docs/cmek-integrations.md %}) |
| **SC-28(1)** | Cryptographic protection at rest | Envelope model with per-resource DEKs | [Architecture]({% link docs/architecture.md %}) |
| **AC-3** | Access enforcement | IAM at key scope; org policy constraints | `keys get-iam-policy` |
| **AC-5** | Separation of duties | `cloudkms.admin` excludes crypto use, by design | SoD test transcript; SCC `KMS_ROLE_SEPARATION` |
| **AC-6** | Least privilege | Key-scoped bindings; custom role without destroy | IAM export |
| **AC-17** | Remote access | Private Service Connect for EKM; VPC-SC | Network configuration |
| **AU-2 / AU-3** | Audit events and content | Data Access logs with caller, IP, resource | BigQuery queries |
| **AU-9** | Protection of audit information | Locked retention on the archive bucket; Admin Activity cannot be disabled | `buckets describe --format="value(retentionPolicy)"` |
| **AU-12** | Audit record generation | Org-level audit config; org sink with `--include-children` | `get-iam-policy --format="json(auditConfigs)"` |
| **CM-3** | Configuration change control | Terraform + PR review + drift detection | PR history; drift issues |
| **CP-9** | System backup | Configuration export; Google-managed key durability | Backup archive |
| **IA-5** | Authenticator management | Workload Identity Federation; no service account keys | Org policy |
| **SI-4** | System monitoring | Canaries, log-based alerts, SCC, asset feeds | [Monitoring]({% link docs/monitoring.md %}) |

### ISO/IEC 27001:2022 Annex A

| Control | Title | How this build satisfies it |
|:--|:--|:--|
| **A.5.15** | Access control | IAM at key scope, reviewed quarterly |
| **A.5.16** | Identity management | Cloud Identity; workload identity; no SA keys |
| **A.5.17** | Authentication information | Federation everywhere; nothing to store |
| **A.5.18** | Access rights | Quarterly key access review with Policy Analyzer |
| **A.5.23** | Information security for cloud services | Documented protection-level selection |
| **A.8.10** | Information deletion | Version destruction with a 30-day window and checklist |
| **A.8.12** | Data leakage prevention | Key-scoped IAM; AAD binding; VPC-SC |
| **A.8.24** | **Use of cryptography** | The cryptographic policy this guide implements |
| **A.8.15** | Logging | Data Access logs to BigQuery with a locked archive |
| **A.8.16** | Monitoring activities | Decrypt-anomaly detection; canaries; SCC |
| **A.8.32** | Change management | CI/CD with review, policy gate, and drift detection |

{: .note }
> **A.8.24 expects a documented cryptographic policy**, not just working
> encryption. It should state: approved algorithms and key lengths, key generation
> requirements, cryptoperiods per key purpose, roles and responsibilities, and key
> destruction procedures. Sections
> [1]({% link docs/overview.md %}), [7.1]({% link docs/rotation.md %}), and
> [9]({% link docs/backup-dr.md %}) map to those clauses directly.

### SOC 2 Trust Services Criteria

| Criterion | Requirement | Implementation | Evidence artifact |
|:--|:--|:--|:--|
| **CC6.1** | Logical access controls | Key-scoped IAM; org policy constraints | IAM export |
| **CC6.3** | Role-based least privilege | `cloudkms.admin` cannot decrypt; custom role cannot destroy | SoD transcript |
| **CC6.6** | Controls over external access | No public members; `allowedPolicyMemberDomains` | SCC findings |
| **CC6.7** | Restricted transmission | TLS by default; PSC for EKM | Configuration |
| **CC7.1** | Detection of configuration changes | Asset feeds, drift detection, log alerts | Alert policies |
| **CC7.2** | Monitoring for anomalies | Decrypt-volume anomaly query | Saved BigQuery query |
| **CC7.3** | Evaluation of security events | On-call runbook with defined actions | [Monitoring runbook]({% link docs/monitoring.md %}#the-on-call-runbook) |
| **CC8.1** | Change management | PR review, CODEOWNERS, environment approval | PR history |
| **A1.2** | Recovery | Project lien; version restore; config backup | DR test record |

### PCI DSS v4.0

| Requirement | Text (summarized) | Implementation |
|:--|:--|:--|
| **3.5.1** | PAN rendered unreadable wherever stored | Cloud HSM key; application-level encryption or direct `Encrypt` |
| **3.6.1** | Key management procedures documented and implemented | This guide, adopted as your procedure |
| **3.6.1.1** | Procedures cover generation, distribution, storage, changes, retirement | Sections 5, 7.1, 7.2, 9 |
| **3.6.1.2** | Secret and private keys stored in the fewest possible locations | Keys never leave the KMS boundary |
| **3.6.1.3** | Access restricted to the fewest custodians | Key-scoped IAM; PAM for elevation |
| **3.6.1.4** | Keys stored in a secure form | FIPS 140-2 L3 HSM; no export API |
| **3.7.1** | Key generation using strong cryptography | AES-256, RSA-3072+, P-384 in an HSM |
| **3.7.2** | Secure key distribution | Keys are never distributed — operations occur in KMS |
| **3.7.3** | Secure key storage | Same |
| **3.7.4** | Key changes at the end of the cryptoperiod | 90-day rotation with a documented cryptoperiod |
| **3.7.5** | Retirement or replacement of keys | Version disable → destroy with a 30-day window |
| **3.7.6** | Manual clear-text operations use split knowledge and dual control | **Not applicable** for KMS-generated keys; the BYOK ceremony satisfies it where used |
| **3.7.7** | Unauthorized substitution prevented | Immutable protection level; org policy; `prevent_destroy` |
| **3.7.8** | Key custodians acknowledge their responsibilities | Ceremony log signatures (BYOK) |
| **10.2** | Audit logs for all access | Data Access logs |

{: .important }
> **PCI DSS 3.7.6 — split knowledge and dual control — applies to *manual
> clear-text key operations*.** If you never handle clear-text key material (pure
> Cloud KMS, keys generated in the HSM), the requirement is largely satisfied by
> design and you should say so explicitly, citing the absence of any export API and
> the key attestation as evidence. If you do BYOK, the two-custodian ceremony in
> [7.2]({% link docs/import-byok.md %}) is what satisfies it — and the ceremony log
> is the evidence. A ceremony performed by one person, however careful, does not.

## What is genuinely different about Google Cloud

Worth stating explicitly in a control narrative, because assessors familiar with
AWS or Azure will ask:

| Property | Consequence for the control narrative |
|:--|:--|
| **`roles/cloudkms.admin` excludes crypto operations by default** | Separation of duties is structural, not constructed. Point at the built-in role definition |
| **Keys and key rings cannot be deleted** | "Unauthorized key deletion" is not a risk that exists. Only version destruction is |
| **Cloud HSM requires no cluster, ceremony, or backup** | The FIPS 140-2 L3 control has a far smaller operational surface than on AWS or Azure |
| **Key attestations are customer-visible** | You can prove hardware generation cryptographically |
| **Ciphertext carries its version** | Rotation is transparent; there is no "which version wrapped this?" bookkeeping |
| **No cross-region key replication** | Cross-region DR must be designed explicitly. Say how |
| **Data Access logs are off by default** | "We enabled them at the organization on \<date\>" is a required statement |
| **AAD is not logged** | Per-tenant attribution must come from your application logs |

## Evidence collection

```bash
#!/usr/bin/env bash
# scripts/collect_audit_evidence.sh — one command, one evidence pack
set -euo pipefail

ORG_ID="${ORG_ID:?set ORG_ID}"
KEY_PROJECT="${KEY_PROJECT:-contoso-keymgmt-prod}"
LOG_PROJECT="${LOG_PROJECT:-contoso-logging-prod}"
LOCATIONS="${LOCATIONS:-us-east1 us-west1 us europe-west4 global}"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
OUT="audit-evidence-${STAMP}"
mkdir -p "$OUT"

echo "== 1. Key inventory with findings =="
python scripts/key_inventory.py "$KEY_PROJECT" > "$OUT/inventory-summary.txt" 2>&1 || true
mv kms-inventory-*.{json,csv} "$OUT/" 2>/dev/null || true

echo "== 2. Key and key ring configuration =="
mkdir -p "$OUT/config"
for LOC in $LOCATIONS; do
  for RING in $(gcloud kms keyrings list --location="$LOC" --project="$KEY_PROJECT" \
                  --format="value(name.basename())" 2>/dev/null); do
    gcloud kms keys list --keyring="$RING" --location="$LOC" --project="$KEY_PROJECT" \
      --format=json > "$OUT/config/keys-${LOC}-${RING}.json"
    gcloud kms keyrings get-iam-policy "$RING" --location="$LOC" \
      --project="$KEY_PROJECT" --format=json \
      > "$OUT/config/ring-iam-${LOC}-${RING}.json"
    for KEY in $(gcloud kms keys list --keyring="$RING" --location="$LOC" \
                   --project="$KEY_PROJECT" --format="value(name.basename())"); do
      gcloud kms keys get-iam-policy "$KEY" --keyring="$RING" --location="$LOC" \
        --project="$KEY_PROJECT" --format=json \
        > "$OUT/config/key-iam-${LOC}-${RING}-${KEY}.json"
      gcloud kms keys versions list --key="$KEY" --keyring="$RING" \
        --location="$LOC" --project="$KEY_PROJECT" --format=json \
        > "$OUT/config/versions-${LOC}-${RING}-${KEY}.json"
    done
  done
done

echo "== 3. HSM attestations =="
bash scripts/collect_attestations.sh >/dev/null 2>&1 || true
mv hsm-attestations-*.tar.gz "$OUT/" 2>/dev/null || true

echo "== 4. Separation of duties proof =="
{
  echo "Test: a principal with roles/cloudkms.admin cannot encrypt"
  echo -n test | gcloud kms encrypt --key=gcs-cmek-hsm --keyring=prod-platform \
    --location=us-east1 --project="$KEY_PROJECT" \
    --plaintext-file=- --ciphertext-file=/dev/null 2>&1 | tail -3
  echo
  echo "Test: basic roles on the key project (should be empty)"
  gcloud projects get-iam-policy "$KEY_PROJECT" \
    --flatten="bindings[].members" \
    --filter="bindings.role:(roles/owner OR roles/editor)" \
    --format="table(bindings.role, bindings.members)"
} > "$OUT/sod-test.txt" 2>&1

echo "== 5. Organization policy constraints =="
gcloud org-policies list --organization="$ORG_ID" --format=json \
  > "$OUT/org-policies.json"
for C in gcp.restrictNonCmekServices gcp.restrictCmekCryptoKeyProjects \
         cloudkms.allowedProtectionLevels iam.disableServiceAccountKeyCreation; do
  gcloud org-policies describe "$C" --organization="$ORG_ID" --format=json \
    > "$OUT/org-policy-${C}.json" 2>/dev/null || true
done

echo "== 6. Audit logging configuration =="
gcloud organizations get-iam-policy "$ORG_ID" --format="json(auditConfigs)" \
  > "$OUT/audit-configs.json"
gcloud logging sinks list --organization="$ORG_ID" --format=json \
  > "$OUT/log-sinks.json"

echo "== 7. Archive retention lock =="
gcloud storage buckets describe gs://contoso-audit-archive \
  --format="json(retentionPolicy)" > "$OUT/archive-retention.json" 2>/dev/null || true

echo "== 8. Project lien =="
gcloud alpha resource-manager liens list --project="$KEY_PROJECT" --format=json \
  > "$OUT/project-liens.json" 2>/dev/null || true

echo "== 9. Security Command Center findings =="
gcloud scc findings list "organizations/${ORG_ID}" \
  --filter='category:"KMS_" AND state="ACTIVE"' --format=json \
  > "$OUT/scc-kms-findings.json" 2>/dev/null || true

echo "== 10. Key usage over the audit period =="
bq query --project_id="$LOG_PROJECT" --format=prettyjson --nouse_legacy_sql '
  SELECT
    protopayload_auditlog.resourceName                      AS key_name,
    protopayload_auditlog.authenticationInfo.principalEmail AS principal,
    COUNT(*)                                                AS operations,
    MAX(timestamp)                                          AS last_used
  FROM `'"$LOG_PROJECT"'.cloud_audit_logs.cloudaudit_googleapis_com_data_access`
  WHERE protopayload_auditlog.serviceName = "cloudkms.googleapis.com"
    AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY 1, 2 ORDER BY operations DESC LIMIT 500' \
  > "$OUT/key-usage-90d.json" 2>/dev/null || true

echo "== 11. CMEK coverage =="
gcloud asset search-all-resources --scope="organizations/${ORG_ID}" \
  --asset-types="storage.googleapis.com/Bucket,bigquery.googleapis.com/Dataset,sqladmin.googleapis.com/Instance" \
  --format=json > "$OUT/cmek-coverage.json" 2>/dev/null || true

tar czf "${OUT}.tar.gz" "$OUT" && rm -rf "$OUT"
sha256sum "${OUT}.tar.gz" | tee "${OUT}.tar.gz.sha256"
echo "Evidence pack: ${OUT}.tar.gz"
```

{: .tip }
> **Hash the evidence pack and record the hash separately** — in a ticket, an email
> to the auditor, or a signed commit. It lets you demonstrate later that the
> evidence provided in March is byte-identical to what is in your archive, which
> turns a pile of JSON into something with chain of custody.

## The key management policy document

| Section | Content | Source in this guide |
|:--|:--|:--|
| Scope | Which projects, data classes, and locations | [Overview]({% link docs/overview.md %}) |
| Approved algorithms | AES-256-GCM, RSA-3072+, EC P-384, HMAC-SHA-256 | [Overview]({% link docs/overview.md %}#key-purposes-and-algorithms) |
| Key generation | In Cloud HSM (`protection_level = HSM`) only | [Cloud HSM]({% link docs/cloud-hsm.md %}) |
| Key hierarchy | CMEK wraps per-resource DEKs | [Architecture]({% link docs/architecture.md %}) |
| Cryptoperiods | Per key purpose, with justification | [Rotation]({% link docs/rotation.md %}#rotation-policy--what-to-actually-write-down) |
| Roles | Key admin, crypto user, auditor, break-glass, BYOK custodian | [IAM]({% link docs/iam.md %}) |
| Access review | Quarterly, using Policy Analyzer with group expansion | [Access review]({% link docs/iam.md %}#the-access-review) |
| Backup and recovery | Configuration export; project lien; version restore | [Backup &amp; DR]({% link docs/backup-dr.md %}) |
| Destruction | 30-day window, checklist, break-glass only | [Destruction checklist]({% link docs/backup-dr.md %}#the-destruction-checklist) |
| Incident response | Disable → investigate → rotate → re-encrypt | [Monitoring runbook]({% link docs/monitoring.md %}#the-on-call-runbook) |
| Exceptions | Org policy exceptions at folder scope, with review dates | [Policy as Code]({% link docs/policy-as-code.md %}) |

## Common audit findings, and how this build avoids them

| Finding | Root cause | Prevented by |
|:--|:--|:--|
| "No key usage audit trail" | Data Access logs never enabled | Org-level audit config + a health-check alert |
| "Keys are software-protected" | `--protection-level` omitted | `cloudkms.allowedProtectionLevels` + OPA gate |
| "Key rotation not configured" | Not set at creation | Custom org constraint + OPA gate |
| "Key administrators can decrypt" | `roles/owner` or `roles/editor` on the key project | Basic-role check + SCC `KMS_PROJECT_HAS_OWNER` |
| "Same principal administers and uses keys" | Convenience grant | SCC `KMS_ROLE_SEPARATION` + inventory check |
| "CMEK not enforced" | No org policy | `gcp.restrictNonCmekServices` |
| "Workload teams create their own keys" | No project restriction | `gcp.restrictCmekCryptoKeyProjects` |
| "24-hour destroy window" | API default accepted | Custom constraint requiring 30 days |
| "No documented cryptoperiod" | Rotation set to a default with no rationale | Written rotation policy with justification |
| "Service account keys in use" | Not disabled | `iam.disableServiceAccountKeyCreation` |
| "Cross-region DR untested" | No key replication, so nobody tested it | Documented DR pattern + annual exercise |
| "Shadow keys outside IaC" | Console experiments that became permanent | Alert on non-pipeline `CreateCryptoKey` |

---

[Next: 15. Verification &amp; Runbook]({% link docs/verification.md %}){: .btn .btn-primary }
