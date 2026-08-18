---
title: 5.2 gcloud
parent: 5. Creating Keys
nav_order: 2
---

# Creating Keys with `gcloud`
{: .no_toc }

The operational path — fast, scriptable, and the only sane tool during an
incident.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The full build

```bash
gcloud config set project contoso-keymgmt-prod
LOCATION=us-east1
KEYRING=prod-platform
```

### Step 1 — Create the key ring

```bash
gcloud kms keyrings create "$KEYRING" --location="$LOCATION"

gcloud kms keyrings list --location="$LOCATION" \
  --format="table(name.basename(), createTime)"
```

```text
NAME           CREATE_TIME
prod-platform  2026-08-18T18:20:04Z
```

{: .warning }
> **There is no `gcloud kms keyrings delete`.** The command does not exist,
> because the API does not support it. If you typo the name, the typo is permanent
> for the life of the project. Check the name before you press enter.

### Step 2 — Create the key

```bash
gcloud kms keys create gcs-cmek-hsm \
  --keyring="$KEYRING" \
  --location="$LOCATION" \
  --purpose=encryption \
  --protection-level=hsm \
  --default-algorithm=google-symmetric-encryption \
  --rotation-period=90d \
  --next-rotation-time="$(date -u -d '+90 days' +%Y-%m-%dT%H:%M:%SZ)" \
  --destroy-scheduled-duration=30d \
  --labels=environment=prod,data-class=confidential,owner=platform-team,cost-center=cc-4417,managed-by=gcloud
```

```bash
gcloud kms keys describe gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION" \
  --format="yaml(name,purpose,versionTemplate,rotationPeriod,nextRotationTime,destroyScheduledDuration)"
```

```yaml
destroyScheduledDuration: 2592000s
name: projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform/cryptoKeys/gcs-cmek-hsm
nextRotationTime: '2026-11-16T18:22:31.000000Z'
purpose: ENCRYPT_DECRYPT
rotationPeriod: 7776000s
versionTemplate:
  algorithm: GOOGLE_SYMMETRIC_ENCRYPTION
  protectionLevel: HSM
```

### Parameter reference

| Flag | Values | Notes |
|:--|:--|:--|
| `--purpose` | `encryption`, `asymmetric-signing`, `asymmetric-encryption`, `mac`, `raw-encryption` | **Immutable** |
| `--protection-level` | `software`, `hsm`, `external`, `external-vpc` | **Immutable** |
| `--default-algorithm` | See the table below | Changeable for future versions on symmetric keys |
| `--rotation-period` | e.g. `90d`, `7776000s` | **Symmetric `encryption` keys only.** Changeable |
| `--next-rotation-time` | RFC 3339 timestamp | Required with `--rotation-period` |
| `--destroy-scheduled-duration` | `24h` to `120d` | **Set it deliberately;** the default is 24 hours |
| `--skip-initial-version-creation` | flag | Required for `external` and for import |
| `--labels` | `k=v,k=v` | Lowercase only |
| `--import-only` | flag | The key can only ever receive imported versions |

### Algorithms by purpose

```bash
# The authoritative list for your CLI version
gcloud kms keys create --help | sed -n '/--default-algorithm/,/^ *--/p' | head -40
```

| Purpose | `--default-algorithm` |
|:--|:--|
| `encryption` | `google-symmetric-encryption` |
| `asymmetric-signing` | `rsa-sign-pss-2048-sha256`, `rsa-sign-pss-3072-sha256`, `rsa-sign-pss-4096-sha256`, `rsa-sign-pss-4096-sha512`, `rsa-sign-pkcs1-*`, `ec-sign-p256-sha256`, `ec-sign-p384-sha384`, `ec-sign-secp256k1-sha256` |
| `asymmetric-encryption` | `rsa-decrypt-oaep-2048-sha256`, `...-3072-sha256`, `...-4096-sha256`, `...-4096-sha512` |
| `mac` | `hmac-sha256`, `hmac-sha1`, `hmac-sha224`, `hmac-sha384`, `hmac-sha512` |
| `raw-encryption` | `aes-128-gcm`, `aes-256-gcm`, `aes-128-cbc`, `aes-256-cbc`, `aes-128-ctr`, `aes-256-ctr` |

```bash
# Signing key
gcloud kms keys create artifact-signing-hsm \
  --keyring=prod-signing --location="$LOCATION" \
  --purpose=asymmetric-signing \
  --protection-level=hsm \
  --default-algorithm=ec-sign-p384-sha384 \
  --destroy-scheduled-duration=30d \
  --labels=environment=prod,data-class=internal,managed-by=gcloud

# MAC key
gcloud kms keys create token-mac-hsm \
  --keyring=prod-payments --location="$LOCATION" \
  --purpose=mac \
  --protection-level=hsm \
  --default-algorithm=hmac-sha256 \
  --destroy-scheduled-duration=30d \
  --labels=environment=prod,data-class=confidential,managed-by=gcloud
```

{: .warning }
> **`--rotation-period` is only valid for symmetric `encryption` keys.** Passing
> it with `--purpose=asymmetric-signing` or `--purpose=mac` fails with an
> `INVALID_ARGUMENT`. That is not an arbitrary restriction: rotating an asymmetric
> key silently would break every verifier that trusts the old public key.

### Step 3 — Verify end to end

```bash
KEY_ARGS="--key=gcs-cmek-hsm --keyring=$KEYRING --location=$LOCATION"

echo -n "gcloud-canary" \
  | gcloud kms encrypt $KEY_ARGS --plaintext-file=- --ciphertext-file=/tmp/ct.bin

gcloud kms decrypt $KEY_ARGS --ciphertext-file=/tmp/ct.bin --plaintext-file=-
echo
```

```text
gcloud-canary
```

```bash
# With additional authenticated data — and prove it binds
echo -n "tenant=acme-corp" > /tmp/aad.bin

echo -n "sensitive" | gcloud kms encrypt $KEY_ARGS \
  --plaintext-file=- --ciphertext-file=/tmp/ct-aad.bin \
  --additional-authenticated-data-file=/tmp/aad.bin

# Correct AAD — succeeds
gcloud kms decrypt $KEY_ARGS --ciphertext-file=/tmp/ct-aad.bin \
  --plaintext-file=- --additional-authenticated-data-file=/tmp/aad.bin; echo

# Wrong AAD — MUST fail
echo -n "tenant=evil-corp" > /tmp/aad-wrong.bin
gcloud kms decrypt $KEY_ARGS --ciphertext-file=/tmp/ct-aad.bin \
  --plaintext-file=- --additional-authenticated-data-file=/tmp/aad-wrong.bin 2>&1 | tail -2

rm -f /tmp/ct.bin /tmp/ct-aad.bin /tmp/aad.bin /tmp/aad-wrong.bin
```

```text
ERROR: (gcloud.kms.decrypt) FAILED_PRECONDITION: Decryption failed:
the ciphertext is invalid.
```

{: .important }
> **Run that failure case and keep the output.** It is the proof that AAD is
> cryptographically binding, which is the control you point an auditor at when
> explaining per-tenant isolation.

## A complete, idempotent provisioning script

```bash
#!/usr/bin/env bash
# scripts/provision_keys.sh — idempotent Cloud KMS provisioning
set -euo pipefail

PROJECT="${PROJECT:-contoso-keymgmt-prod}"
LOCATION="${LOCATION:-us-east1}"
ENVIRONMENT="${ENVIRONMENT:-prod}"
PROTECTION="${PROTECTION:-hsm}"
ROTATION="${ROTATION:-90d}"
DESTROY_WINDOW="${DESTROY_WINDOW:-30d}"

log() { printf '[%s] %s\n' "$(date -u +%H:%M:%SZ)" "$*"; }

ensure_keyring () {
  local RING="$1"
  if gcloud kms keyrings describe "$RING" --location="$LOCATION" \
       --project="$PROJECT" >/dev/null 2>&1; then
    log "key ring $RING exists"
  else
    log "creating key ring $RING"
    gcloud kms keyrings create "$RING" --location="$LOCATION" --project="$PROJECT"
  fi
}

ensure_key () {                # ensure_key <ring> <key> <purpose> <algorithm> <data-class>
  local RING="$1" KEY="$2" PURPOSE="$3" ALGO="$4" CLASS="$5"

  if gcloud kms keys describe "$KEY" --keyring="$RING" --location="$LOCATION" \
       --project="$PROJECT" >/dev/null 2>&1; then
    log "key $RING/$KEY exists"
    return
  fi

  local ARGS=(
    --keyring="$RING" --location="$LOCATION" --project="$PROJECT"
    --purpose="$PURPOSE"
    --protection-level="$PROTECTION"
    --default-algorithm="$ALGO"
    --destroy-scheduled-duration="$DESTROY_WINDOW"
    --labels="environment=${ENVIRONMENT},data-class=${CLASS},owner=platform-team,managed-by=gcloud"
  )

  # Rotation applies only to symmetric encryption keys
  if [ "$PURPOSE" = "encryption" ]; then
    ARGS+=(--rotation-period="$ROTATION"
           --next-rotation-time="$(date -u -d "+${ROTATION%d} days" +%Y-%m-%dT%H:%M:%SZ)")
  fi

  log "creating key $RING/$KEY ($PURPOSE / $ALGO / $PROTECTION)"
  gcloud kms keys create "$KEY" "${ARGS[@]}"
}

# --- Platform key ring ------------------------------------------------------
ensure_keyring prod-platform
ensure_key prod-platform gcs-cmek-hsm     encryption google-symmetric-encryption confidential
ensure_key prod-platform bq-cmek-hsm      encryption google-symmetric-encryption confidential
ensure_key prod-platform disk-cmek-hsm    encryption google-symmetric-encryption confidential
ensure_key prod-platform sql-cmek-hsm     encryption google-symmetric-encryption confidential
ensure_key prod-platform secrets-cmek-hsm encryption google-symmetric-encryption confidential

# --- Payments key ring (separate IAM boundary) ------------------------------
ensure_keyring prod-payments
ensure_key prod-payments pci-chd-hsm       encryption google-symmetric-encryption restricted
ensure_key prod-payments pci-token-mac-hsm mac        hmac-sha256                 restricted

# --- Signing key ring -------------------------------------------------------
ensure_keyring prod-signing
ensure_key prod-signing artifact-signing-hsm asymmetric-signing ec-sign-p384-sha384 internal

log "Done"
gcloud kms keys list --keyring=prod-platform --location="$LOCATION" \
  --project="$PROJECT" \
  --format="table(name.basename(), purpose, versionTemplate.protectionLevel, rotationPeriod, nextRotationTime)"
```

```bash
chmod +x scripts/provision_keys.sh
./scripts/provision_keys.sh
```

```text
[18:20:04Z] key ring prod-platform exists
[18:20:05Z] creating key prod-platform/gcs-cmek-hsm (encryption / google-symmetric-encryption / hsm)
[18:20:09Z] creating key prod-platform/bq-cmek-hsm (encryption / google-symmetric-encryption / hsm)
...
NAME              PURPOSE          PROTECTION_LEVEL  ROTATION_PERIOD  NEXT_ROTATION_TIME
gcs-cmek-hsm      ENCRYPT_DECRYPT  HSM               7776000s         2026-11-16T18:22:31Z
bq-cmek-hsm       ENCRYPT_DECRYPT  HSM               7776000s         2026-11-16T18:22:39Z
disk-cmek-hsm     ENCRYPT_DECRYPT  HSM               7776000s         2026-11-16T18:22:44Z
```

{: .tip }
> **The `if PURPOSE = encryption` branch is not defensive padding.** Without it,
> the script fails on the MAC and signing keys with `INVALID_ARGUMENT` on
> `--rotation-period`, and a partially-created key ring is permanent. Filter
> first, act second.

## Useful day-two operations

```bash
# Every key ring and key in a location
for RING in $(gcloud kms keyrings list --location="$LOCATION" \
                --format="value(name.basename())"); do
  echo "== $RING =="
  gcloud kms keys list --keyring="$RING" --location="$LOCATION" \
    --format="table(name.basename(), purpose, versionTemplate.protectionLevel, rotationPeriod, primary.state)"
done

# Versions of a key, with state
gcloud kms keys versions list --key=gcs-cmek-hsm \
  --keyring="$KEYRING" --location="$LOCATION" \
  --format="table(name.basename(), state, createTime, destroyTime)"

# Who can use this key?
gcloud kms keys get-iam-policy gcs-cmek-hsm \
  --keyring="$KEYRING" --location="$LOCATION" \
  --format="table(bindings.role, bindings.members)"

# Rotate on demand — creates a new version and makes it primary
gcloud kms keys versions create --key=gcs-cmek-hsm \
  --keyring="$KEYRING" --location="$LOCATION" --primary

# Disable a version — the containment action (breaks decrypt too)
gcloud kms keys versions disable 2 --key=gcs-cmek-hsm \
  --keyring="$KEYRING" --location="$LOCATION"

# Re-enable
gcloud kms keys versions enable 2 --key=gcs-cmek-hsm \
  --keyring="$KEYRING" --location="$LOCATION"

# Change the rotation period
gcloud kms keys update gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION" \
  --rotation-period=180d \
  --next-rotation-time="$(date -u -d '+180 days' +%Y-%m-%dT%H:%M:%SZ)"

# Remove rotation entirely
gcloud kms keys update gcs-cmek-hsm --keyring="$KEYRING" --location="$LOCATION" \
  --remove-rotation-schedule
```

{: .warning }
> **`gcloud kms keys versions disable` breaks decryption, not just encryption.**
> On Cloud KMS a disabled version cannot decrypt either — so disabling the primary
> version of a CMEK immediately breaks every bucket, dataset, and disk using it.
> That is the point when you are containing an incident, and a serious outage when
> you are not. Know the blast radius first; the dependency map is in
> [Verification]({% link docs/verification.md %}).

## Asymmetric and MAC operations

```bash
SIGN_ARGS="--key=artifact-signing-hsm --keyring=prod-signing --location=$LOCATION"

# Retrieve the public key — this is the only key material that ever leaves
gcloud kms keys versions get-public-key 1 $SIGN_ARGS --output-file=/tmp/pub.pem
openssl ec -pubin -in /tmp/pub.pem -text -noout | head -4

# Sign a digest (the API signs a digest, not the message)
echo -n "artifact-sha256:9f2c1a" | openssl dgst -sha384 -binary > /tmp/digest.bin
gcloud kms asymmetric-sign --version=1 $SIGN_ARGS \
  --digest-algorithm=sha384 --input-file=/tmp/digest.bin \
  --signature-file=/tmp/sig.bin

# Verify locally with the public key — no KMS call needed
openssl dgst -sha384 -verify /tmp/pub.pem -signature /tmp/sig.bin \
  <(echo -n "artifact-sha256:9f2c1a")

# MAC
MAC_ARGS="--key=pci-token-mac-hsm --keyring=prod-payments --location=$LOCATION"
echo -n "session-token-abc123" > /tmp/token.txt
gcloud kms mac-sign --version=1 $MAC_ARGS \
  --input-file=/tmp/token.txt --signature-file=/tmp/mac.bin

gcloud kms mac-verify --version=1 $MAC_ARGS \
  --input-file=/tmp/token.txt --signature-file=/tmp/mac.bin

rm -f /tmp/pub.pem /tmp/digest.bin /tmp/sig.bin /tmp/token.txt /tmp/mac.bin
```

```text
Verified OK
success: true
```

{: .tip }
> **Signature verification does not need Cloud KMS.** `get-public-key` gives you a
> PEM you can distribute and verify against locally, forever — no API call, no
> quota, no latency, and it keeps working if KMS is unreachable. Only *signing*
> requires the service. Design your verifiers accordingly.

---

[Next: 5.3 Terraform]({% link docs/kms-terraform.md %}){: .btn .btn-primary }
