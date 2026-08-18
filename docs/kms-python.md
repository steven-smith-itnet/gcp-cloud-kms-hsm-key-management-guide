---
title: 5.5 Python client
parent: 5. Creating Keys
nav_order: 5
---

# Creating and Using Keys with the Python Client
{: .no_toc }

Use the client library when keys are created *by an application* — per-tenant
keys, keys minted during onboarding — and to produce the audit evidence a
compliance process actually consumes.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## A production-shaped key provisioner

`scripts/kms_provision.py`

```python
#!/usr/bin/env python3
"""Idempotent Cloud KMS key ring and key provisioning.

Creates resources only when they do not already exist, so the script is safe to
re-run from a pipeline. Handles the purpose/algorithm/rotation interactions that
trip up hand-rolled scripts.

Note: key rings and keys CANNOT be deleted. Every create is permanent.
"""
from __future__ import annotations

import argparse
import json
import logging
import sys
from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone

from google.api_core import exceptions as gexc
from google.api_core import retry
from google.cloud import kms
from google.protobuf import duration_pb2, timestamp_pb2

log = logging.getLogger("kms-provision")

PURPOSES = {
    "encryption": kms.CryptoKey.CryptoKeyPurpose.ENCRYPT_DECRYPT,
    "asymmetric-signing": kms.CryptoKey.CryptoKeyPurpose.ASYMMETRIC_SIGN,
    "asymmetric-encryption": kms.CryptoKey.CryptoKeyPurpose.ASYMMETRIC_DECRYPT,
    "mac": kms.CryptoKey.CryptoKeyPurpose.MAC,
    "raw-encryption": kms.CryptoKey.CryptoKeyPurpose.RAW_ENCRYPT_DECRYPT,
}

ALGORITHMS = {
    "google-symmetric-encryption":
        kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm.GOOGLE_SYMMETRIC_ENCRYPTION,
    "ec-sign-p256-sha256":
        kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm.EC_SIGN_P256_SHA256,
    "ec-sign-p384-sha384":
        kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm.EC_SIGN_P384_SHA384,
    "rsa-sign-pss-3072-sha256":
        kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm.RSA_SIGN_PSS_3072_SHA256,
    "rsa-sign-pss-4096-sha512":
        kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm.RSA_SIGN_PSS_4096_SHA512,
    "rsa-decrypt-oaep-3072-sha256":
        kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm.RSA_DECRYPT_OAEP_3072_SHA256,
    "hmac-sha256":
        kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm.HMAC_SHA256,
    "aes-256-gcm":
        kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm.AES_256_GCM,
}

PROTECTION = {
    "software": kms.ProtectionLevel.SOFTWARE,
    "hsm": kms.ProtectionLevel.HSM,
    "external": kms.ProtectionLevel.EXTERNAL,
    "external-vpc": kms.ProtectionLevel.EXTERNAL_VPC,
}


@dataclass
class KeySpecification:
    key_ring: str
    name: str
    purpose: str = "encryption"
    algorithm: str = "google-symmetric-encryption"
    protection_level: str = "hsm"
    rotation_days: int | None = 90
    destroy_scheduled_days: int = 30
    labels: dict[str, str] = field(default_factory=dict)

    @property
    def rotation_supported(self) -> bool:
        # Automatic rotation applies only to symmetric ENCRYPT_DECRYPT keys.
        return self.purpose == "encryption"


class KeyProvisioner:
    def __init__(self, project_id: str, location: str) -> None:
        self.client = kms.KeyManagementServiceClient()
        self.project_id = project_id
        self.location = location
        self.location_path = self.client.common_location_path(project_id, location)

    # -- key ring ---------------------------------------------------------
    def ensure_key_ring(self, name: str) -> str:
        ring_path = self.client.key_ring_path(self.project_id, self.location, name)
        try:
            self.client.get_key_ring(name=ring_path)
            log.info("key ring %s exists", name)
            return ring_path
        except gexc.NotFound:
            pass

        log.info("creating key ring %s in %s", name, self.location)
        ring = self.client.create_key_ring(
            request={
                "parent": self.location_path,
                "key_ring_id": name,
                "key_ring": {},
            },
            retry=retry.Retry(deadline=120.0),
        )
        return ring.name

    # -- key --------------------------------------------------------------
    def ensure_key(self, spec: KeySpecification) -> dict:
        ring_path = self.client.key_ring_path(
            self.project_id, self.location, spec.key_ring
        )
        key_path = self.client.crypto_key_path(
            self.project_id, self.location, spec.key_ring, spec.name
        )

        try:
            self.client.get_crypto_key(name=key_path)
            log.info("key %s/%s exists", spec.key_ring, spec.name)
            return self.describe(spec.key_ring, spec.name)
        except gexc.NotFound:
            pass

        crypto_key: dict = {
            "purpose": PURPOSES[spec.purpose],
            "version_template": {
                "algorithm": ALGORITHMS[spec.algorithm],
                "protection_level": PROTECTION[spec.protection_level],
            },
            "labels": {**spec.labels, "managed-by": "python-client"},
            "destroy_scheduled_duration": duration_pb2.Duration(
                seconds=spec.destroy_scheduled_days * 86400
            ),
        }

        if spec.rotation_supported and spec.rotation_days:
            crypto_key["rotation_period"] = duration_pb2.Duration(
                seconds=spec.rotation_days * 86400
            )
            next_rotation = datetime.now(timezone.utc) + timedelta(days=spec.rotation_days)
            ts = timestamp_pb2.Timestamp()
            ts.FromDatetime(next_rotation)
            crypto_key["next_rotation_time"] = ts
        elif spec.rotation_days and not spec.rotation_supported:
            log.info("rotation not supported for purpose=%s — skipping", spec.purpose)

        log.info("creating key %s/%s (%s / %s / %s)",
                 spec.key_ring, spec.name, spec.purpose,
                 spec.algorithm, spec.protection_level)

        self.client.create_crypto_key(
            request={
                "parent": ring_path,
                "crypto_key_id": spec.name,
                "crypto_key": crypto_key,
            },
            retry=retry.Retry(deadline=300.0),
        )
        return self.describe(spec.key_ring, spec.name)

    # -- describe ---------------------------------------------------------
    def describe(self, key_ring: str, name: str) -> dict:
        key_path = self.client.crypto_key_path(
            self.project_id, self.location, key_ring, name
        )
        key = self.client.get_crypto_key(name=key_path)

        versions = [
            {
                "version": v.name.rsplit("/", 1)[-1],
                "state": kms.CryptoKeyVersion.CryptoKeyVersionState(v.state).name,
                "protection_level": kms.ProtectionLevel(v.protection_level).name,
                "algorithm": kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm(
                    v.algorithm).name,
                "create_time": v.create_time.rfc3339() if v.create_time else None,
            }
            for v in self.client.list_crypto_key_versions(parent=key_path)
        ]

        return {
            "name": key.name,
            "purpose": kms.CryptoKey.CryptoKeyPurpose(key.purpose).name,
            "protection_level": kms.ProtectionLevel(
                key.version_template.protection_level).name,
            "algorithm": kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm(
                key.version_template.algorithm).name,
            "rotation_period_days": (
                key.rotation_period.seconds // 86400 if key.rotation_period else None
            ),
            "next_rotation_time": (
                key.next_rotation_time.rfc3339() if key.next_rotation_time else None
            ),
            "destroy_scheduled_duration_days": (
                key.destroy_scheduled_duration.seconds // 86400
                if key.destroy_scheduled_duration else None
            ),
            "primary_version": (
                key.primary.name.rsplit("/", 1)[-1] if key.primary.name else None
            ),
            "labels": dict(key.labels),
            "versions": versions,
        }


def main() -> int:
    ap = argparse.ArgumentParser(description="Provision Cloud KMS keys")
    ap.add_argument("--project", required=True)
    ap.add_argument("--location", default="us-east1")
    ap.add_argument("--key-ring", required=True)
    ap.add_argument("--key", required=True)
    ap.add_argument("--purpose", default="encryption", choices=sorted(PURPOSES))
    ap.add_argument("--algorithm", default="google-symmetric-encryption",
                    choices=sorted(ALGORITHMS))
    ap.add_argument("--protection-level", default="hsm", choices=sorted(PROTECTION))
    ap.add_argument("--rotation-days", type=int, default=90)
    ap.add_argument("--destroy-days", type=int, default=30)
    ap.add_argument("--label", action="append", default=[], metavar="key=value")
    args = ap.parse_args()

    logging.basicConfig(level=logging.INFO,
                        format="%(asctime)s %(levelname)-7s %(message)s")

    prov = KeyProvisioner(args.project, args.location)
    prov.ensure_key_ring(args.key_ring)

    spec = KeySpecification(
        key_ring=args.key_ring,
        name=args.key,
        purpose=args.purpose,
        algorithm=args.algorithm,
        protection_level=args.protection_level,
        rotation_days=args.rotation_days,
        destroy_scheduled_days=args.destroy_days,
        labels=dict(l.split("=", 1) for l in args.label),
    )

    try:
        print(json.dumps(prov.ensure_key(spec), indent=2))
    except gexc.GoogleAPICallError as exc:
        log.error("Cloud KMS error: %s", exc.message)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

```bash
python scripts/kms_provision.py \
  --project contoso-keymgmt-prod --location us-east1 \
  --key-ring prod-platform --key gcs-cmek-hsm \
  --protection-level hsm --rotation-days 90 \
  --label environment=prod --label data-class=confidential
```

```json
{
  "name": "projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform/cryptoKeys/gcs-cmek-hsm",
  "purpose": "ENCRYPT_DECRYPT",
  "protection_level": "HSM",
  "algorithm": "GOOGLE_SYMMETRIC_ENCRYPTION",
  "rotation_period_days": 90,
  "next_rotation_time": "2026-11-16T18:22:31Z",
  "destroy_scheduled_duration_days": 30,
  "primary_version": "1",
  "labels": { "environment": "prod", "data-class": "confidential", "managed-by": "python-client" },
  "versions": [
    {
      "version": "1",
      "state": "ENABLED",
      "protection_level": "HSM",
      "algorithm": "GOOGLE_SYMMETRIC_ENCRYPTION",
      "create_time": "2026-08-18T18:22:29Z"
    }
  ]
}
```

{: .warning }
> **Every `ensure_key` that actually creates something is permanent.** Unlike the
> AWS and Azure equivalents of this script, there is no cleanup path. Run it
> against a sandbox *project* while you are developing it, and delete the project
> when you are done.

## Cryptographic operations

```python
#!/usr/bin/env python3
"""Encrypt, decrypt, sign, and MAC with Cloud KMS — with CRC32C integrity."""
import crcmod
from google.cloud import kms

client = kms.KeyManagementServiceClient()

KEY = client.crypto_key_path(
    "contoso-keymgmt-prod", "us-east1", "prod-platform", "gcs-cmek-hsm"
)


def crc32c(data: bytes) -> int:
    fn = crcmod.predefined.mkPredefinedCrcFun("crc-32c")
    return fn(data)


def encrypt(plaintext: bytes, aad: bytes = b"") -> bytes:
    """Encrypt with the key's PRIMARY version. You cannot choose the version."""
    response = client.encrypt(
        request={
            "name": KEY,
            "plaintext": plaintext,
            "additional_authenticated_data": aad,
            # Integrity: the service verifies these and echoes verification flags
            "plaintext_crc32c": crc32c(plaintext),
            "additional_authenticated_data_crc32c": crc32c(aad) if aad else None,
        }
    )

    # Verify the response was not corrupted in transit
    if not response.verified_plaintext_crc32c:
        raise RuntimeError("Request corrupted in transit: plaintext CRC mismatch")
    if response.ciphertext_crc32c != crc32c(response.ciphertext):
        raise RuntimeError("Response corrupted in transit: ciphertext CRC mismatch")

    return response.ciphertext


def decrypt(ciphertext: bytes, aad: bytes = b"") -> bytes:
    """Decrypt. KMS selects the version from the ciphertext — you never specify it."""
    response = client.decrypt(
        request={
            "name": KEY,
            "ciphertext": ciphertext,
            "additional_authenticated_data": aad,
            "ciphertext_crc32c": crc32c(ciphertext),
            "additional_authenticated_data_crc32c": crc32c(aad) if aad else None,
        }
    )
    if response.plaintext_crc32c != crc32c(response.plaintext):
        raise RuntimeError("Response corrupted in transit: plaintext CRC mismatch")
    return response.plaintext


if __name__ == "__main__":
    aad = b"tenant=acme-corp"
    ct = encrypt(b"4111111111111111", aad)
    assert decrypt(ct, aad) == b"4111111111111111"
    print("round trip: OK")

    # Prove the AAD binds
    try:
        decrypt(ct, b"tenant=evil-corp")
        raise SystemExit("SECURITY FAILURE: AAD was not enforced")
    except Exception as exc:                      # noqa: BLE001
        print("AAD binding:", type(exc).__name__, "(expected)")
```

{: .important }
> **The CRC32C fields are not optional ceremony.** Cloud KMS validates them
> server-side and returns verification flags; checking those flags is how you
> detect corruption between your process and the service. Google's own samples
> include them for exactly this reason. Omitting them means a corrupted request
> silently produces a ciphertext you cannot decrypt.

### Asymmetric signing

```python
import hashlib

from google.cloud import kms

client = kms.KeyManagementServiceClient()

VERSION = client.crypto_key_version_path(
    "contoso-keymgmt-prod", "us-east1", "prod-signing",
    "artifact-signing-hsm", "1",
)

message = b"artifact-sha256:9f2c1a...e4"
digest = hashlib.sha384(message).digest()

response = client.asymmetric_sign(
    request={
        "name": VERSION,
        "digest": {"sha384": digest},
        "digest_crc32c": crc32c(digest),
    }
)
if not response.verified_digest_crc32c:
    raise RuntimeError("digest corrupted in transit")

signature = response.signature
print("signature:", signature.hex()[:64], "...")

# Verify locally — no KMS call, no quota, works offline
public_key = client.get_public_key(request={"name": VERSION})

from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import ec, utils

pub = serialization.load_pem_public_key(public_key.pem.encode())
pub.verify(signature, message, ec.ECDSA(hashes.SHA384()))
print("verified locally: OK")
```

{: .tip }
> **Distribute the PEM and verify locally.** Verification never needs Cloud KMS,
> which means your verifiers keep working during a KMS outage, consume no quota,
> and add no latency. Only signing requires the service.

### MAC

```python
VERSION = client.crypto_key_version_path(
    "contoso-keymgmt-prod", "us-east1", "prod-payments",
    "pci-token-mac-hsm", "1",
)

data = b"session-token-abc123"

mac = client.mac_sign(
    request={"name": VERSION, "data": data, "data_crc32c": crc32c(data)}
)

verify = client.mac_verify(
    request={
        "name": VERSION,
        "data": data,
        "mac": mac.mac,
        "data_crc32c": crc32c(data),
        "mac_crc32c": crc32c(mac.mac),
    }
)
assert verify.success
print("MAC verified in-HSM: OK")
```

## The audit evidence exporter

`scripts/key_inventory.py`

```python
#!/usr/bin/env python3
"""Export a Cloud KMS inventory across projects and locations as audit evidence."""
from __future__ import annotations

import csv
import json
import sys
from datetime import datetime, timezone

from google.api_core import exceptions as gexc
from google.cloud import kms
from google.cloud import resourcemanager_v3

REQUIRED_LABELS = {"environment", "data-class", "owner", "managed-by"}
MAX_ROTATION_DAYS = 365
LOCATIONS = ["us-east1", "us-west1", "us", "europe-west4", "global"]


def audit_project(project_id: str) -> list[dict]:
    client = kms.KeyManagementServiceClient()
    rows: list[dict] = []
    now = datetime.now(timezone.utc)

    for location in LOCATIONS:
        parent = client.common_location_path(project_id, location)
        try:
            key_rings = list(client.list_key_rings(parent=parent))
        except (gexc.PermissionDenied, gexc.NotFound, gexc.InvalidArgument):
            continue

        for ring in key_rings:
            ring_name = ring.name.rsplit("/", 1)[-1]
            try:
                keys = list(client.list_crypto_keys(parent=ring.name))
            except gexc.PermissionDenied:
                rows.append(_row(project_id, location, ring_name, None,
                                 ["NO_ACCESS_TO_KEYS"]))
                continue

            # Ring-level IAM
            try:
                ring_policy = client.get_iam_policy(request={"resource": ring.name})
                ring_members = sorted({
                    m for b in ring_policy.bindings for m in b.members
                })
            except gexc.PermissionDenied:
                ring_members = []

            for key in keys:
                findings: list[str] = []
                key_name = key.name.rsplit("/", 1)[-1]

                protection = kms.ProtectionLevel(
                    key.version_template.protection_level).name
                purpose = kms.CryptoKey.CryptoKeyPurpose(key.purpose).name

                if protection == "SOFTWARE":
                    findings.append("SOFTWARE_PROTECTION")

                rotation_days = (
                    key.rotation_period.seconds // 86400
                    if key.rotation_period else None
                )
                if purpose == "ENCRYPT_DECRYPT":
                    if rotation_days is None:
                        findings.append("NO_ROTATION")
                    elif rotation_days > MAX_ROTATION_DAYS:
                        findings.append(f"ROTATION_TOO_SLOW_{rotation_days}D")

                destroy_days = (
                    key.destroy_scheduled_duration.seconds // 86400
                    if key.destroy_scheduled_duration else 1
                )
                if destroy_days < 7:
                    findings.append(f"SHORT_DESTROY_WINDOW_{destroy_days}D")

                missing = REQUIRED_LABELS - set(key.labels)
                if missing:
                    findings.append(f"MISSING_LABELS:{'|'.join(sorted(missing))}")

                # Versions
                versions = list(client.list_crypto_key_versions(parent=key.name))
                states = [
                    kms.CryptoKeyVersion.CryptoKeyVersionState(v.state).name
                    for v in versions
                ]
                enabled = states.count("ENABLED")
                if enabled == 0:
                    findings.append("NO_ENABLED_VERSIONS")
                if "DESTROY_SCHEDULED" in states:
                    findings.append("VERSION_PENDING_DESTRUCTION")

                # Key-level IAM
                try:
                    policy = client.get_iam_policy(request={"resource": key.name})
                    members = sorted({m for b in policy.bindings for m in b.members})
                except gexc.PermissionDenied:
                    members = []

                if any(m in ("allUsers", "allAuthenticatedUsers") for m in members):
                    findings.append("PUBLIC_ACCESS")

                # A principal with both admin and crypto-use roles
                roles_by_member: dict[str, set[str]] = {}
                try:
                    for b in policy.bindings:
                        for m in b.members:
                            roles_by_member.setdefault(m, set()).add(b.role)
                    for m, roles in roles_by_member.items():
                        if "roles/cloudkms.admin" in roles and any(
                            r.startswith("roles/cloudkms.crypto") or
                            r in ("roles/cloudkms.signer", "roles/cloudkms.signerVerifier")
                            for r in roles
                        ):
                            findings.append(f"ROLE_SEPARATION_VIOLATION:{m}")
                except NameError:
                    pass

                age_days = (now - key.create_time.replace(tzinfo=timezone.utc)).days \
                    if key.create_time else None

                rows.append({
                    "Project": project_id,
                    "Location": location,
                    "KeyRing": ring_name,
                    "Key": key_name,
                    "FullName": key.name,
                    "Purpose": purpose,
                    "ProtectionLevel": protection,
                    "Algorithm": kms.CryptoKeyVersion.CryptoKeyVersionAlgorithm(
                        key.version_template.algorithm).name,
                    "RotationDays": rotation_days or "",
                    "NextRotation": (
                        key.next_rotation_time.rfc3339()
                        if key.next_rotation_time else ""
                    ),
                    "DestroyWindowDays": destroy_days,
                    "PrimaryVersion": (
                        key.primary.name.rsplit("/", 1)[-1] if key.primary.name else ""
                    ),
                    "VersionCount": len(versions),
                    "EnabledVersions": enabled,
                    "AgeDays": age_days if age_days is not None else "",
                    "Labels": json.dumps(dict(key.labels)),
                    "KeyMembers": ";".join(members),
                    "RingMembers": ";".join(ring_members),
                    "Findings": ";".join(findings) if findings else "NONE",
                })
    return rows


def _row(project, location, ring, key, findings) -> dict:
    return {
        "Project": project, "Location": location, "KeyRing": ring,
        "Key": key or "<none>", "FullName": "", "Purpose": "",
        "ProtectionLevel": "", "Algorithm": "", "RotationDays": "",
        "NextRotation": "", "DestroyWindowDays": "", "PrimaryVersion": "",
        "VersionCount": 0, "EnabledVersions": 0, "AgeDays": "",
        "Labels": "{}", "KeyMembers": "", "RingMembers": "",
        "Findings": ";".join(findings),
    }


def main() -> int:
    projects = sys.argv[1:]
    if not projects:
        client = resourcemanager_v3.ProjectsClient()
        projects = [
            p.project_id for p in client.search_projects(query="state:ACTIVE")
        ]

    rows: list[dict] = []
    for p in projects:
        print(f"Scanning {p} ...", file=sys.stderr)
        rows.extend(audit_project(p))

    stamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
    with open(f"kms-inventory-{stamp}.json", "w") as fh:
        json.dump(rows, fh, indent=2, default=str)
    if rows:
        with open(f"kms-inventory-{stamp}.csv", "w", newline="") as fh:
            w = csv.DictWriter(fh, fieldnames=list(rows[0]))
            w.writeheader()
            w.writerows(rows)

    flagged = [r for r in rows if r["Findings"] != "NONE"]
    print(f"\n{len(rows)} keys across {len(projects)} project(s); "
          f"{len(flagged)} with findings")
    for r in flagged:
        print(f"  [{r['Location']}] {r['KeyRing']}/{r['Key']:<24} {r['Findings']}")
    return 1 if flagged else 0


if __name__ == "__main__":
    sys.exit(main())
```

```bash
python scripts/key_inventory.py contoso-keymgmt-prod contoso-app-prod
```

```text
Scanning contoso-keymgmt-prod ...
Scanning contoso-app-prod ...

19 keys across 2 project(s); 4 with findings
  [us-east1] dev-scratch/test-key             SOFTWARE_PROTECTION;NO_ROTATION;MISSING_LABELS:data-class|owner
  [us-east1] prod-platform/legacy-import      NO_ROTATION;SHORT_DESTROY_WINDOW_1D
  [global]   old-ring/experiment-2024         SOFTWARE_PROTECTION;NO_ENABLED_VERSIONS;MISSING_LABELS:data-class|environment|owner
  [us-east1] prod-payments/pci-chd-hsm        ROLE_SEPARATION_VIOLATION:group:gcp-key-admins@contoso.com
```

{: .tip }
> **`ROLE_SEPARATION_VIOLATION` is the finding worth acting on immediately.** It
> means a principal holds both `roles/cloudkms.admin` and a crypto-use role on the
> same key, which collapses the separation of duties GCP gives you by default.
> Security Command Center's `KMS_ROLE_SEPARATION` detector finds the same thing —
> having both is a useful cross-check.

## Error handling that matters

```python
from google.api_core import exceptions as gexc


def robust_decrypt(client, key_name: str, ciphertext: bytes, aad: bytes = b"") -> bytes:
    try:
        return client.decrypt(request={
            "name": key_name, "ciphertext": ciphertext,
            "additional_authenticated_data": aad,
        }).plaintext

    except gexc.PermissionDenied as exc:
        # The caller lacks cloudkms.cryptoKeyVersions.useToDecrypt, or the
        # key is in a project the caller cannot see at all.
        raise PermissionError(
            f"Not authorized to decrypt with {key_name}. "
            "Check roles/cloudkms.cryptoKeyEncrypterDecrypter at the key scope."
        ) from exc

    except gexc.FailedPrecondition as exc:
        # Almost always: the key VERSION that produced this ciphertext is
        # DISABLED or DESTROYED, or the AAD does not match.
        raise RuntimeError(
            "Decryption failed. The key version may be disabled or destroyed, "
            f"or the additional authenticated data does not match. AAD={aad!r}"
        ) from exc

    except gexc.NotFound as exc:
        raise RuntimeError(f"Key not found: {key_name}") from exc

    except gexc.InvalidArgument as exc:
        # Ciphertext malformed, or produced by a different key.
        raise RuntimeError(
            "Ciphertext is invalid or was produced by a different key"
        ) from exc

    except gexc.ResourceExhausted:
        # Per-project KMS request quota. Let the client's retry policy handle it.
        raise

    except gexc.ServiceUnavailable:
        raise
```

{: .warning }
> **`FAILED_PRECONDITION` on decrypt is the most misdiagnosed Cloud KMS error.**
> It almost never means the ciphertext is corrupt. It means either the version
> that produced it is disabled or destroyed, or the AAD does not byte-for-byte
> match. Log the AAD on both sides and check the version states before you go
> looking for storage corruption:
>
> ```bash
> gcloud kms keys versions list --key=gcs-cmek-hsm \
>   --keyring=prod-platform --location=us-east1 \
>   --format="table(name.basename(), state)"
> ```

## Throttling and retries

```python
from google.api_core import retry
from google.cloud import kms

client = kms.KeyManagementServiceClient()

kms_retry = retry.Retry(
    initial=1.0,
    maximum=60.0,
    multiplier=2.0,
    deadline=300.0,
    predicate=retry.if_exception_type(
        gexc.ServiceUnavailable,
        gexc.DeadlineExceeded,
        gexc.ResourceExhausted,   # quota
        gexc.InternalServerError,
    ),
)

response = client.encrypt(
    request={"name": KEY, "plaintext": b"payload"},
    retry=kms_retry,
    timeout=30.0,
)
```

{: .tip }
> **Cloud KMS quota is per-project and per-region, and HSM operations have a lower
> limit than software ones.** A workload that calls `Encrypt` per record will hit
> `RESOURCE_EXHAUSTED` under load. The answer is the same on every cloud: use the
> envelope pattern so one KMS call serves many records, and cache the DEK for a
> bounded, documented period. See
> [Envelope Encryption]({% link docs/envelope-encryption.md %}).

---

[Next: 6. IAM &amp; Access Control]({% link docs/iam.md %}){: .btn .btn-primary }
