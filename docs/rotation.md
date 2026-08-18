---
title: 7.1 Rotation
parent: 7. Key Operations
nav_order: 1
---

# Key Rotation
{: .no_toc }

Cloud KMS rotation is the simplest of the three major clouds — and the one whose
limits are most often misdescribed in a control narrative.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What rotation actually does

```text
BEFORE rotation                        AFTER rotation
┌────────────────────────────┐        ┌────────────────────────────┐
│ Key: gcs-cmek-hsm          │        │ Key: gcs-cmek-hsm          │
│                            │        │                            │
│ v1: ENABLED — PRIMARY      │        │ v1: ENABLED (retained,     │
│                            │        │     still decrypts)        │
│                            │        │ v2: ENABLED — PRIMARY      │
└────────────────────────────┘        └────────────────────────────┘

New encrypts use the primary.  Existing ciphertext is UNCHANGED and still
                               decrypts, because it names v1 internally.
```

| What changes | What does **not** change |
|:--|:--|
| A new version is created and becomes primary | The key name or resource path |
| New `Encrypt` calls use the new version | Existing ciphertext — nothing is re-encrypted |
| | Old versions — they stay `ENABLED` and keep decrypting |
| | IAM bindings, labels, or the rotation schedule |

{: .important }
> **Automatic rotation does not re-encrypt your data.** Data encrypted three
> rotations ago is still protected by the three-rotations-ago version. If your
> requirement is "no data is protected by key material older than N months" — which
> is what some readings of PCI DSS and internal standards ask for — then rotation
> alone does not satisfy it. You need re-encryption, covered below. Get this
> straight before you write it into a control description.

## Automatic rotation

```bash
LOCATION=us-east1
KEYRING=prod-platform

gcloud kms keys update gcs-cmek-hsm \
  --keyring="$KEYRING" --location="$LOCATION" \
  --rotation-period=90d \
  --next-rotation-time="$(date -u -d '+90 days' +%Y-%m-%dT%H:%M:%SZ)"

gcloud kms keys describe gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION" \
  --format="yaml(rotationPeriod,nextRotationTime,primary.name,primary.state)"
```

```yaml
nextRotationTime: '2026-11-16T18:22:31.000000Z'
primary:
  name: projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform/cryptoKeys/gcs-cmek-hsm/cryptoKeyVersions/1
  state: ENABLED
rotationPeriod: 7776000s
```

| Constraint | Value |
|:--|:--|
| Minimum rotation period | 24 hours |
| Maximum rotation period | 100 years |
| Applies to | **Symmetric `ENCRYPT_DECRYPT` keys only** |
| `next-rotation-time` | Required alongside `--rotation-period` |

### Where automatic rotation is not available

| Key purpose | Automatic rotation | Alternative |
|:--|:--|:--|
| `ENCRYPT_DECRYPT` (symmetric) | ✅ Supported | — |
| `ASYMMETRIC_SIGN` | ❌ Not supported | Manual version creation + verifier coordination |
| `ASYMMETRIC_DECRYPT` | ❌ Not supported | Manual version creation |
| `MAC` | ❌ Not supported | Manual, with dual-key acceptance during overlap |
| `RAW_ENCRYPT_DECRYPT` | ❌ Not supported | Manual |
| Imported (BYOK) key versions | ❌ Not supported | Re-import |
| `EXTERNAL` / `EXTERNAL_VPC` | ❌ Not supported | Rotate in your external key manager |

```bash
# Attempting it on a signing key
gcloud kms keys update artifact-signing-hsm \
  --keyring=prod-signing --location="$LOCATION" \
  --rotation-period=90d --next-rotation-time="$(date -u -d '+90 days' +%Y-%m-%dT%H:%M:%SZ)" 2>&1 | tail -2
```

```text
ERROR: (gcloud.kms.keys.update) INVALID_ARGUMENT: Rotation is only supported for
symmetric encryption keys.
```

{: .note }
> **Signing keys are the awkward case, and the restriction is a feature.**
> Rotating a signing key means every verifier must trust the new public key
> *before* you sign with it, and must keep trusting the old one until every
> signature made with it has expired. That is a coordinated release, not a
> scheduled job — which is exactly why Cloud KMS refuses to do it silently.

## Manual rotation

```bash
# Create a new version and make it primary in one step
gcloud kms keys versions create \
  --key=gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION" --primary

gcloud kms keys versions list --key=gcs-cmek-hsm \
  --keyring="$KEYRING" --location="$LOCATION" \
  --format="table(name.basename(), state, createTime)"
```

```text
NAME  STATE    CREATE_TIME
1     ENABLED  2026-08-18T18:22:29Z
2     ENABLED  2026-11-16T09:04:11Z
```

```bash
# Or create without promoting, then promote deliberately after verification
gcloud kms keys versions create \
  --key=gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION"

gcloud kms keys update gcs-cmek-hsm \
  --keyring="$KEYRING" --location="$LOCATION" --primary-version=3
```

{: .tip }
> **Splitting create and promote is the safer pattern for a high-value key.**
> Create the version, verify it is `ENABLED` and that a test encrypt/decrypt
> against it works, and only then make it primary. For an HSM key the new version
> is briefly `PENDING_GENERATION`; promoting it in the same breath occasionally
> races.

## Retiring an old version

```bash
OLD=1

# 1. Confirm nothing still needs it — see the BigQuery query below

# 2. Disable it (reversible). NOTE: this breaks DECRYPT for that version too.
gcloud kms keys versions disable "$OLD" \
  --key=gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION"

# 3. Soak for a full business cycle, watching for failures

# 4. Schedule destruction — reversible for destroy_scheduled_duration
gcloud kms keys versions destroy "$OLD" \
  --key=gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION"

gcloud kms keys versions describe "$OLD" \
  --key=gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION" \
  --format="yaml(state,destroyTime,destroyEventTime)"
```

```yaml
destroyEventTime: '2026-12-16T09:12:04.000000Z'
destroyTime: '2026-12-16T09:12:04.000000Z'
state: DESTROY_SCHEDULED
```

```bash
# Cancel — restores the version to DISABLED, not ENABLED
gcloud kms keys versions restore "$OLD" \
  --key=gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION"

gcloud kms keys versions enable "$OLD" \
  --key=gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION"
```

{: .important }
> **`versions restore` returns the version to `DISABLED`, not `ENABLED`.** People
> cancel a destruction, breathe out, and forget the second command — then spend an
> hour debugging why decryption still fails. Always follow a restore with an
> `enable` and a round-trip test.

{: .warning }
> **`versions disable` breaks decryption immediately.** Unlike some platforms
> where disabling only stops new encryption, a disabled Cloud KMS version cannot
> decrypt. Every object encrypted under it becomes unreadable the moment you press
> enter. That is correct for incident containment and catastrophic otherwise —
> which is precisely why the disable-and-soak step exists before destruction.

### Finding out what still uses a version

```sql
-- BigQuery over the audit log sink from Organization Setup
SELECT
  JSON_VALUE(protopayload_auditlog.requestJson, '$.name')  AS key_or_version,
  protopayload_auditlog.methodName                          AS method,
  protopayload_auditlog.authenticationInfo.principalEmail   AS principal,
  COUNT(*)                                                  AS operations,
  MAX(timestamp)                                            AS last_used
FROM `contoso-logging-prod.cloud_audit_logs.cloudaudit_googleapis_com_data_access`
WHERE protopayload_auditlog.serviceName = 'cloudkms.googleapis.com'
  AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND JSON_VALUE(protopayload_auditlog.requestJson, '$.name')
      LIKE '%/cryptoKeys/gcs-cmek-hsm%'
GROUP BY 1, 2, 3
ORDER BY operations DESC;
```

{: .warning }
> **Cloud KMS Data Access logs record the *key* on `Decrypt`, not the version.**
> The service selects the version from the ciphertext, so the request does not name
> one — which means you cannot answer "is anything still using version 1?" from the
> audit log alone. The reliable test is behavioral: **disable the version and watch
> for errors** for a full business cycle. That is why the soak period is not
> optional here, and why 30 days is a floor rather than a formality.

## Re-encryption — when rotation is not enough

If your requirement is genuinely "retire the old key material," you must
re-encrypt. Cloud KMS has no `ReEncrypt` API, so this is a decrypt-then-encrypt
loop.

```python
#!/usr/bin/env python3
"""Re-encrypt ciphertext under the current primary version.

Rotation does NOT re-encrypt. This does. Run it after rotation for any dataset
whose control requires retiring old key material.
"""
from __future__ import annotations

import crcmod
from google.cloud import kms

client = kms.KeyManagementServiceClient()
KEY = client.crypto_key_path(
    "contoso-keymgmt-prod", "us-east1", "prod-platform", "gcs-cmek-hsm"
)


def _crc32c(data: bytes) -> int:
    return crcmod.predefined.mkPredefinedCrcFun("crc-32c")(data)


def reencrypt(ciphertext: bytes, aad: bytes = b"") -> bytes:
    """Decrypt with whichever version produced it, re-encrypt with the primary."""
    plaintext = client.decrypt(request={
        "name": KEY,
        "ciphertext": ciphertext,
        "additional_authenticated_data": aad,
        "ciphertext_crc32c": _crc32c(ciphertext),
    }).plaintext
    try:
        return client.encrypt(request={
            "name": KEY,
            "plaintext": plaintext,
            "additional_authenticated_data": aad,
            "plaintext_crc32c": _crc32c(plaintext),
        }).ciphertext
    finally:
        del plaintext


if __name__ == "__main__":
    # Iterate your own datastore; this is the shape, not the storage layer.
    # for record in store.scan():
    #     store.update(record.id, blob=reencrypt(record.blob, record.aad))
    key = client.get_crypto_key(name=KEY)
    print("primary version:", key.primary.name.rsplit("/", 1)[-1])
```

{: .warning }
> **Re-encryption briefly holds plaintext in your process.** That is unavoidable
> without a `ReEncrypt` API, and it is a real difference from AWS KMS, where
> `ReEncrypt` keeps the plaintext inside the service boundary. Run re-encryption
> jobs on hardened, short-lived infrastructure, and consider whether the
> re-encryption requirement is worth that exposure — for many designs, retaining
> old versions is the safer answer.

### Service-managed re-encryption

For CMEK, "re-encryption" means asking the service to re-wrap its DEK. Most
services do it automatically when the key's primary version changes — but on
their own schedule.

| Service | Behavior after rotation |
|:--|:--|
| Cloud Storage | New objects use the new version. **Existing objects keep their version** — rewrite to re-wrap |
| BigQuery | Re-wraps table DEKs in the background |
| Persistent Disk | Re-wraps on the next attach, or via a snapshot/restore cycle |
| Cloud SQL | Re-wraps on the next instance restart or maintenance |
| Secret Manager | New secret versions use the new key version |

```bash
# GCS: rewrite objects to re-wrap them under the current primary
gcloud storage objects update "gs://contoso-prod-data/**" \
  --encryption-key="projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform/cryptoKeys/gcs-cmek-hsm"

# Verify which key version an object is using
gcloud storage objects describe gs://contoso-prod-data/report.parquet \
  --format="value(kms_key)"
```

```text
projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform/cryptoKeys/gcs-cmek-hsm/cryptoKeyVersions/1
```

{: .tip }
> **`gcloud storage objects describe --format="value(kms_key)"` returns the
> *version*.** That is the only reliable way to find objects still wrapped under an
> old version, and it is what you run before disabling one. Script it across the
> bucket and treat any object still on the old version as a blocker.

## Rotation coverage report

```bash
#!/usr/bin/env bash
# scripts/rotation_report.sh — audit and remediate rotation coverage
set -euo pipefail

PROJECT="${PROJECT:-contoso-keymgmt-prod}"
LOCATIONS="${LOCATIONS:-us-east1 us-west1 us europe-west4 global}"
FIX="${FIX:-false}"
PERIOD="${PERIOD:-90d}"
MAX_DAYS="${MAX_DAYS:-365}"

printf '%-12s %-18s %-24s %-18s %-10s %-12s %s\n' \
  LOCATION KEYRING KEY PURPOSE PROT ROTATION ACTION
printf '%.0s-' {1..108}; echo

for LOCATION in $LOCATIONS; do
  RINGS=$(gcloud kms keyrings list --location="$LOCATION" --project="$PROJECT" \
    --format="value(name.basename())" 2>/dev/null) || continue

  for RING in $RINGS; do
    while IFS=$'\t' read -r KEY PURPOSE PROT ROTATION; do
      [ -z "$KEY" ] && continue
      ACTION="-"

      if [ "$PURPOSE" != "ENCRYPT_DECRYPT" ]; then
        ACTION="N/A"
      elif [ -z "$ROTATION" ] || [ "$ROTATION" = "null" ]; then
        if [ "$FIX" = "true" ]; then
          gcloud kms keys update "$KEY" --keyring="$RING" --location="$LOCATION" \
            --project="$PROJECT" --rotation-period="$PERIOD" \
            --next-rotation-time="$(date -u -d "+${PERIOD%d} days" +%Y-%m-%dT%H:%M:%SZ)" \
            >/dev/null 2>&1 && ACTION="ENABLED" || ACTION="FAILED"
        else
          ACTION="NEEDS-ROTATION"
        fi
      else
        DAYS=$(( ${ROTATION%s} / 86400 ))
        [ "$DAYS" -gt "$MAX_DAYS" ] && ACTION="TOO-SLOW-${DAYS}D"
      fi

      printf '%-12s %-18s %-24s %-18s %-10s %-12s %s\n' \
        "$LOCATION" "$RING" "$KEY" "$PURPOSE" "$PROT" "${ROTATION:--}" "$ACTION"
    done < <(gcloud kms keys list --keyring="$RING" --location="$LOCATION" \
               --project="$PROJECT" \
               --format="value[separator='\t'](name.basename(),purpose,versionTemplate.protectionLevel,rotationPeriod)" \
               2>/dev/null)
  done
done
```

```bash
./scripts/rotation_report.sh              # report only
FIX=true ./scripts/rotation_report.sh     # report and remediate
```

```text
LOCATION     KEYRING            KEY                      PURPOSE            PROT       ROTATION     ACTION
------------------------------------------------------------------------------------------------------------
us-east1     prod-platform      gcs-cmek-hsm             ENCRYPT_DECRYPT    HSM        7776000s     -
us-east1     prod-platform      bq-cmek-hsm              ENCRYPT_DECRYPT    HSM        7776000s     -
us-east1     prod-platform      legacy-import            ENCRYPT_DECRYPT    HSM        -            NEEDS-ROTATION
us-east1     prod-payments      pci-token-mac-hsm        MAC                HSM        -            N/A
us-east1     prod-signing       artifact-signing-hsm     ASYMMETRIC_SIGN    HSM        -            N/A
global       old-ring           experiment-2024          ENCRYPT_DECRYPT    SOFTWARE   31536000s    TOO-SLOW-365D
```

{: .tip }
> The `N/A` rows are the interesting part. Naively enabling rotation everywhere
> produces a wall of `INVALID_ARGUMENT` for MAC and asymmetric keys, which then
> hides the genuine `NEEDS-ROTATION` findings. Filter by purpose first.

## Rotation policy — what to actually write down

| Key purpose | Automatic rotation | Cadence | Re-encryption required? |
|:--|:--|:--|:--|
| CMEK for GCS / BigQuery / Compute / SQL | Enabled | 90 days | No — services re-wrap on their own schedule |
| Application envelope keys | Enabled | 90 days | **Yes**, if your control retires old material |
| Signing keys | **Not supported** | Annual, coordinated | N/A — verifiers trust both during overlap |
| MAC keys | **Not supported** | Quarterly, with dual acceptance | N/A |
| Imported (BYOK) | **Not supported** | Per the source system's schedule | Re-import |
| `EXTERNAL` / `EXTERNAL_VPC` | **Not supported** | Rotate in your external key manager | Depends on the EKM |

{: .note }
> **On "annual key rotation" in compliance frameworks.** PCI DSS v4.0 requires
> keys be changed "at the end of their defined cryptoperiod," and lets *you* define
> that period from a documented risk analysis — it does not mandate 12 months.
> Write the cryptoperiod down, justify it, and make rotation match it. Being able
> to show the reasoning is what passes the assessment; a number with no rationale
> behind it is what fails it.

---

[Next: 7.2 Import &amp; BYOK]({% link docs/import-byok.md %}){: .btn .btn-primary }
