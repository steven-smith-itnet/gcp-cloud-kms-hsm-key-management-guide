---
title: 11. Monitoring & Alerting
nav_order: 12
---

# Monitoring &amp; Alerting
{: .no_toc }

**Phase 7 — Observe.** What to watch, what to page on, and what belongs in a
weekly report.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What to alert on

Alert design for key management has an unusual property: the highest-severity
events are **configuration changes**, not performance metrics.

| Signal | Severity | Why |
|:--|:--|:--|
| `DestroyCryptoKeyVersion` | **Page** | Potential permanent data loss |
| Key version disabled in production | **Page** | Immediate outage — disabled versions cannot decrypt |
| Cloud EKM operation failures | **Page** | All external keys unusable now |
| Audit config or log sink modified | **Page** | Someone is covering their tracks |
| KMS canary failure | **Page** | Encryption is actually broken right now |
| Decrypt volume anomaly (> 3σ) | **Page** | Possible bulk exfiltration |
| Project deletion requested on the key project | **Page** | The only wholesale key-loss path |
| `SetIamPolicy` on a key or key ring | Ticket | Access model changed |
| Key created outside the pipeline | Ticket | Shadow key, and permanent |
| HSM quota above 80% | Ticket | Outage ahead |
| Human principal decrypting production data | Ticket | Should be rare and explainable |
| Key without a rotation schedule | Weekly report | Compliance drift |
| Keys unused for 90 days | Monthly report | Retirement candidates |
| Billable key version count growing | Monthly report | Cost and cleanup |

## Notification channels

```bash
PROJECT=contoso-keymgmt-prod

CRITICAL=$(gcloud monitoring channels create \
  --project="$PROJECT" \
  --display-name="KeyMgmt CRITICAL (PagerDuty)" \
  --type=webhook_tokenauth \
  --channel-labels=url=https://events.pagerduty.com/integration/REPLACE/enqueue \
  --format="value(name)")

WARNING=$(gcloud monitoring channels create \
  --project="$PROJECT" \
  --display-name="KeyMgmt WARNING (email)" \
  --type=email \
  --channel-labels=email_address=keymgmt-alerts@contoso.com \
  --format="value(name)")

echo "critical: $CRITICAL"
echo "warning:  $WARNING"
```

## Log-based metrics and alerting policies

Most KMS signals live in the audit log rather than in a native metric, so they
need log-based metrics.

```bash
#!/usr/bin/env bash
# scripts/create_kms_alerts.sh
set -euo pipefail
PROJECT="${PROJECT:-contoso-keymgmt-prod}"
CRITICAL="${CRITICAL:?set CRITICAL notification channel}"
WARNING="${WARNING:?set WARNING notification channel}"

metric () {                # metric <name> <description> <filter>
  gcloud logging metrics create "$1" --project="$PROJECT" \
    --description="$2" --log-filter="$3" 2>/dev/null \
    || gcloud logging metrics update "$1" --project="$PROJECT" \
       --description="$2" --log-filter="$3"
  echo "  metric: $1"
}

metric kms_version_destroy \
  "A key version was scheduled for destruction" \
  'protoPayload.serviceName="cloudkms.googleapis.com"
   AND protoPayload.methodName="DestroyCryptoKeyVersion"'

metric kms_version_disabled \
  "A key version was disabled" \
  'protoPayload.serviceName="cloudkms.googleapis.com"
   AND protoPayload.methodName="UpdateCryptoKeyVersion"
   AND protoPayload.request.cryptoKeyVersion.state="DISABLED"'

metric kms_iam_changed \
  "IAM policy changed on a key or key ring" \
  'protoPayload.serviceName="cloudkms.googleapis.com"
   AND protoPayload.methodName="SetIamPolicy"'

metric kms_key_created_manually \
  "A key was created by something other than the CI service account" \
  'protoPayload.serviceName="cloudkms.googleapis.com"
   AND protoPayload.methodName="CreateCryptoKey"
   AND NOT protoPayload.authenticationInfo.principalEmail="gha-keymgmt@contoso-app-prod.iam.gserviceaccount.com"'

metric kms_permission_denied \
  "Cloud KMS permission denied" \
  'protoPayload.serviceName="cloudkms.googleapis.com"
   AND protoPayload.status.code=7'

metric kms_external_errors \
  "Cloud KMS operations failing against an external key manager" \
  'protoPayload.serviceName="cloudkms.googleapis.com"
   AND severity>=ERROR
   AND (protoPayload.status.message:"external" OR protoPayload.status.message:"EKM")'

metric kms_human_decrypt_prod \
  "A human principal decrypted with a production key" \
  'protoPayload.serviceName="cloudkms.googleapis.com"
   AND protoPayload.methodName:"Decrypt"
   AND NOT protoPayload.authenticationInfo.principalEmail=~"gserviceaccount\.com$"
   AND protoPayload.resourceName:"/prod-"'

metric audit_config_changed \
  "An audit config or log sink was modified" \
  '(protoPayload.methodName="SetIamPolicy"
    AND protoPayload.serviceData.policyDelta.auditConfigDeltas:*)
   OR protoPayload.methodName="google.logging.v2.ConfigServiceV2.DeleteSink"'

policy () {                # policy <display-name> <metric> <channel> <severity> <doc>
  gcloud monitoring policies create --project="$PROJECT" --policy-from-file=- <<YAML
displayName: $1
documentation:
  content: |
    $5
  mimeType: text/markdown
conditions:
  - displayName: $2 events detected
    conditionThreshold:
      filter: >-
        metric.type = "logging.googleapis.com/user/$2"
      comparison: COMPARISON_GT
      thresholdValue: 0
      duration: 0s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_SUM
combiner: OR
severity: $4
notificationChannels:
  - $3
YAML
  echo "  policy: $1"
}

policy "Cloud KMS key version scheduled for destruction" \
  kms_version_destroy "$CRITICAL" CRITICAL \
  "A key version was scheduled for destruction. If unintentional, restore it NOW: gcloud kms keys versions restore V --key=K --keyring=R --location=L, then enable it."

policy "Cloud KMS key version disabled" \
  kms_version_disabled "$CRITICAL" CRITICAL \
  "A key version was disabled. Disabled versions CANNOT DECRYPT — every resource using this version is failing."

policy "Cloud KMS external key manager errors" \
  kms_external_errors "$CRITICAL" CRITICAL \
  "Cloud KMS cannot reach the external key manager. Check the EKM, the network path, and the TLS certificate."

policy "Cloud KMS audit configuration changed" \
  audit_config_changed "$CRITICAL" CRITICAL \
  "An audit config or log sink was modified. Treat as a security incident until proven otherwise."

policy "Cloud KMS IAM policy changed" \
  kms_iam_changed "$WARNING" WARNING \
  "IAM changed on a key or key ring. Confirm it came from the pipeline."

policy "Cloud KMS key created outside the pipeline" \
  kms_key_created_manually "$WARNING" WARNING \
  "A key was created by a principal other than CI. Keys cannot be deleted — investigate."

policy "Human decrypted with a production key" \
  kms_human_decrypt_prod "$WARNING" WARNING \
  "A named user, not a service account, decrypted production data. Confirm it was authorized."
```

{: .important }
> **`kms_key_created_manually` and `audit_config_changed` are the two
> highest-value alerts here.** The first catches shadow keys — which on Google Cloud
> are permanent — and console experiments becoming production. The second catches
> the first move of anyone who intends to use a key quietly, and it works because
> Admin Activity logging cannot be turned off.

## Native metrics

```bash
# What Cloud KMS actually publishes
gcloud monitoring metrics-descriptors list \
  --filter='metric.type:cloudkms.googleapis.com' \
  --format="table(type, displayName, metricKind, valueType)"
```

| Metric | Use |
|:--|:--|
| `serviceruntime.googleapis.com/api/request_count` | Request rate by method and response code |
| `serviceruntime.googleapis.com/api/request_latencies` | Latency — critical for EKM |
| `serviceruntime.googleapis.com/quota/allocation/usage` | Quota headroom |
| `serviceruntime.googleapis.com/quota/exceeded` | Throttling |

```bash
# Quota headroom
gcloud monitoring policies create --project="$PROJECT" --policy-from-file=- <<YAML
displayName: Cloud KMS request quota above 80%
conditions:
  - displayName: quota usage high
    conditionThreshold:
      filter: >-
        resource.type = "consumer_quota"
        AND resource.label.service = "cloudkms.googleapis.com"
        AND metric.type = "serviceruntime.googleapis.com/quota/allocation/usage"
      comparison: COMPARISON_GT
      thresholdValue: 0.8
      duration: 300s
combiner: OR
severity: WARNING
notificationChannels:
  - ${WARNING}
YAML

# Latency — the EKM early-warning signal
gcloud monitoring policies create --project="$PROJECT" --policy-from-file=- <<YAML
displayName: Cloud KMS p99 latency elevated
conditions:
  - displayName: p99 above 1s
    conditionThreshold:
      filter: >-
        resource.type = "consumed_api"
        AND resource.label.service = "cloudkms.googleapis.com"
        AND metric.type = "serviceruntime.googleapis.com/api/request_latencies"
      comparison: COMPARISON_GT
      thresholdValue: 1000
      duration: 300s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_PERCENTILE_99
combiner: OR
severity: WARNING
notificationChannels:
  - ${WARNING}
YAML
```

{: .tip }
> **Latency is the leading indicator for an EKM problem.** Before an external key
> manager fails outright, it gets slow — and Cloud KMS eventually times out. A p99
> alert gives you minutes of warning that a hard failure alert does not.

## Synthetic canaries

Alerts tell you a metric moved. A canary tells you whether encryption actually
works right now.

```python
# functions/kms_canary/main.py — Cloud Run job, every 5 minutes via Cloud Scheduler
import os
import time

import functions_framework
from google.cloud import kms
from google.cloud import monitoring_v3

PROJECT = os.environ["PROJECT_ID"]
CANARY_KEYS = os.environ["CANARY_KEYS"].split(",")   # full key resource names

kms_client = kms.KeyManagementServiceClient()
metric_client = monitoring_v3.MetricServiceClient()


def emit(metric: str, value: float, labels: dict[str, str]) -> None:
    series = monitoring_v3.TimeSeries()
    series.metric.type = f"custom.googleapis.com/kms/{metric}"
    for k, v in labels.items():
        series.metric.labels[k] = v
    series.resource.type = "global"
    now = time.time()
    point = monitoring_v3.Point({
        "interval": {"end_time": {"seconds": int(now)}},
        "value": {"double_value": value},
    })
    series.points = [point]
    metric_client.create_time_series(
        name=f"projects/{PROJECT}", time_series=[series]
    )


def check(key_name: str) -> tuple[bool, float, str]:
    start = time.perf_counter()
    try:
        ct = kms_client.encrypt(request={
            "name": key_name, "plaintext": b"canary",
        }).ciphertext
        pt = kms_client.decrypt(request={
            "name": key_name, "ciphertext": ct,
        }).plaintext
        elapsed = (time.perf_counter() - start) * 1000
        if pt != b"canary":
            return False, elapsed, "round trip returned wrong plaintext"
        return True, elapsed, ""
    except Exception as exc:                      # noqa: BLE001
        return False, (time.perf_counter() - start) * 1000, \
               f"{type(exc).__name__}: {exc}"


@functions_framework.http
def canary(request):
    failures = []
    for key_name in CANARY_KEYS:
        ok, ms, err = check(key_name)
        short = "/".join(key_name.split("/")[-3:])
        emit("canary_success", 1.0 if ok else 0.0, {"key": short})
        emit("canary_latency_ms", ms, {"key": short})
        if not ok:
            failures.append(f"{short}: {err}")

    if failures:
        # A non-2xx makes the Cloud Scheduler job fail, which is itself alertable
        return ("Canary failures:\n" + "\n".join(failures), 500)
    return (f"OK: {len(CANARY_KEYS)} keys verified", 200)
```

```bash
gcloud scheduler jobs create http kms-canary \
  --project="$PROJECT" --location=us-east1 --schedule="*/5 * * * *" \
  --uri="https://kms-canary-abc123-ue.a.run.app" \
  --oidc-service-account-email="kms-canary@${PROJECT}.iam.gserviceaccount.com" \
  --attempt-deadline=60s

# Alert on failure OR silence
gcloud monitoring policies create --project="$PROJECT" --policy-from-file=- <<YAML
displayName: Cloud KMS canary failing or silent
documentation:
  content: |
    A KMS encrypt/decrypt round trip is failing, or the canary has stopped
    reporting. Check: is the key version ENABLED? Did an IAM binding change?
    Is an external key manager reachable?
conditions:
  - displayName: canary reporting failure
    conditionThreshold:
      filter: >-
        metric.type = "custom.googleapis.com/kms/canary_success"
      comparison: COMPARISON_LT
      thresholdValue: 1
      duration: 300s
  - displayName: canary silent
    conditionAbsent:
      filter: >-
        metric.type = "custom.googleapis.com/kms/canary_success"
      duration: 900s
combiner: OR
severity: CRITICAL
notificationChannels:
  - ${CRITICAL}
YAML
```

{: .important }
> **The `conditionAbsent` clause is not optional.** If the canary stops running —
> a deployment broke it, the service account lost its role, the scheduler was
> disabled — you get no failure signal at all. A canary that fails silently is worse
> than no canary, because it produces false confidence.

## The dashboard

```bash
gcloud monitoring dashboards create --project="$PROJECT" --config-from-file=- <<'YAML'
displayName: Key Management
mosaicLayout:
  columns: 12
  tiles:
    - width: 6
      height: 4
      widget:
        title: Canary success by key
        xyChart:
          dataSets:
            - timeSeriesQuery:
                timeSeriesFilter:
                  filter: metric.type="custom.googleapis.com/kms/canary_success"
                  aggregation:
                    alignmentPeriod: 300s
                    perSeriesAligner: ALIGN_MIN
    - xPos: 6
      width: 6
      height: 4
      widget:
        title: Canary latency p99
        xyChart:
          dataSets:
            - timeSeriesQuery:
                timeSeriesFilter:
                  filter: metric.type="custom.googleapis.com/kms/canary_latency_ms"
                  aggregation:
                    alignmentPeriod: 300s
                    perSeriesAligner: ALIGN_PERCENTILE_99
    - yPos: 4
      width: 6
      height: 4
      widget:
        title: KMS request rate by method
        xyChart:
          dataSets:
            - timeSeriesQuery:
                timeSeriesFilter:
                  filter: >-
                    resource.type="consumed_api"
                    resource.label.service="cloudkms.googleapis.com"
                    metric.type="serviceruntime.googleapis.com/api/request_count"
                  aggregation:
                    alignmentPeriod: 60s
                    perSeriesAligner: ALIGN_RATE
                    crossSeriesReducer: REDUCE_SUM
                    groupByFields: [metric.label.method]
    - xPos: 6
      yPos: 4
      width: 6
      height: 4
      widget:
        title: Permission denied events
        xyChart:
          dataSets:
            - timeSeriesQuery:
                timeSeriesFilter:
                  filter: metric.type="logging.googleapis.com/user/kms_permission_denied"
                  aggregation:
                    alignmentPeriod: 300s
                    perSeriesAligner: ALIGN_SUM
    - yPos: 8
      width: 12
      height: 4
      widget:
        title: Administrative changes
        logsPanel:
          filter: >-
            protoPayload.serviceName="cloudkms.googleapis.com"
            AND (protoPayload.methodName:"Create" OR protoPayload.methodName:"Destroy"
                 OR protoPayload.methodName:"SetIamPolicy" OR protoPayload.methodName:"Update")
          resourceNames:
            - projects/contoso-keymgmt-prod
YAML
```

## The on-call runbook

| Alert | First action | Then |
|:--|:--|:--|
| Key version destroy scheduled | `gcloud kms keys versions describe V --format="value(state,destroyTime)"` | If unauthorized: `versions restore` then `versions enable` **immediately**, then investigate the caller |
| Key version disabled | Identify the version and the caller from the audit log | If unauthorized, `versions enable`; expect dependent services to recover within minutes |
| Canary failing | Check the version state, then the IAM binding, then the network path | Most failures are a disabled version or a removed binding |
| Canary silent | Is the Cloud Scheduler job enabled? Did the canary SA lose its role? | Treat as a canary failure until proven otherwise |
| EKM errors | Is the EKM reachable? Certificate valid? Network path up? | Restore the EKM; there is no Google-side mitigation |
| Audit config changed | Read the `SetIamPolicy` delta: who, when, from where | **Restore the config immediately**, then treat as a security incident |
| IAM policy changed | Compare against the Terraform state | If it did not come from CI, revert and investigate |
| Key created outside the pipeline | Identify the caller and the key | The key is permanent — decide whether to adopt it into IaC or leave it disabled |
| Decrypt volume anomaly | Identify the principal, the key, and the source IP | Correlate with deploys and batch schedules; if unexplained, remove the principal's binding |
| Quota above 80% | Which method and which project? | Enable DEK caching; request a quota increase |
| Project delete requested | **Check for a lien** | `gcloud projects undelete` immediately; add the lien if it was missing |

```bash
#!/usr/bin/env bash
# scripts/incident_snapshot.sh — capture state before it changes
set -euo pipefail
PROJECT="${1:-contoso-keymgmt-prod}"
LOCATION="${2:-us-east1}"
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
OUT="incident-${PROJECT}-${STAMP}"
mkdir -p "$OUT"

gcloud projects get-iam-policy "$PROJECT" --format=json > "$OUT/project-iam.json"

for RING in $(gcloud kms keyrings list --location="$LOCATION" --project="$PROJECT" \
                --format="value(name.basename())"); do
  gcloud kms keys list --keyring="$RING" --location="$LOCATION" --project="$PROJECT" \
    --format=json > "$OUT/keys-${RING}.json"
  for KEY in $(gcloud kms keys list --keyring="$RING" --location="$LOCATION" \
                 --project="$PROJECT" --format="value(name.basename())"); do
    gcloud kms keys versions list --key="$KEY" --keyring="$RING" \
      --location="$LOCATION" --project="$PROJECT" --format=json \
      > "$OUT/versions-${RING}-${KEY}.json"
    gcloud kms keys get-iam-policy "$KEY" --keyring="$RING" \
      --location="$LOCATION" --project="$PROJECT" --format=json \
      > "$OUT/iam-${RING}-${KEY}.json"
  done
done

gcloud logging read '
  protoPayload.serviceName="cloudkms.googleapis.com"
  AND timestamp>="'"$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)"'"' \
  --project="$PROJECT" --limit=1000 --format=json > "$OUT/audit-24h.json"

tar czf "${OUT}.tar.gz" "$OUT" && rm -rf "$OUT"
echo "Snapshot: ${OUT}.tar.gz"
```

---

[Next: 12. CI/CD Automation]({% link docs/cicd.md %}){: .btn .btn-primary }
