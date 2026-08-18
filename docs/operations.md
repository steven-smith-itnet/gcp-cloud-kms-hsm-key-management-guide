---
title: 7. Key Operations
nav_order: 8
has_children: true
---

# Key Operations
{: .no_toc }

**Phase 5 — Operate.** Keys exist and access is scoped. Now put them to work and
keep them healthy through their whole lifecycle.
{: .fs-5 .fw-300 }

---

## The key lifecycle

```mermaid
stateDiagram-v2
    [*] --> ENABLED: create version
    ENABLED --> ENABLED: rotate (new version<br/>becomes primary)
    ENABLED --> DISABLED: disable
    DISABLED --> ENABLED: enable
    ENABLED --> DESTROY_SCHEDULED: destroy
    DISABLED --> DESTROY_SCHEDULED: destroy
    DESTROY_SCHEDULED --> DISABLED: restore
    DESTROY_SCHEDULED --> DESTROYED: window elapses<br/><b>IRREVERSIBLE</b>
```

Two Google-specific behaviors dominate everything in this section:

1. **Ciphertext carries its own version pointer**, so `Decrypt` never needs a
   version hint and rotation is genuinely transparent. The corollary: destroying a
   version silently destroys everything encrypted under it.
2. **Keys and key rings are never deleted.** Only versions are destroyed. A key
   with every version destroyed is a permanent, unusable resource.

## Pages in this section

| Page | What it covers |
|:--|:--|
| [7.1 Rotation]({% link docs/rotation.md %}) | Automatic and manual rotation, version management, and re-encryption |
| [7.2 Import &amp; BYOK]({% link docs/import-byok.md %}) | Import jobs, key wrapping, and the ceremony |
| [7.3 CMEK Integrations]({% link docs/cmek-integrations.md %}) | GCS, BigQuery, Compute, Cloud SQL, GKE, Pub/Sub, Secret Manager |
| [7.4 Envelope Encryption]({% link docs/envelope-encryption.md %}) | DEKs in application code, and Tink |
| [7.5 Autokey]({% link docs/autokey.md %}) | Automatic, per-resource CMEK provisioning |

Backup, recovery, and destruction are covered separately in
[9. Backup, DR &amp; Destruction]({% link docs/backup-dr.md %}), because they span
Cloud KMS and Cloud EKM.
