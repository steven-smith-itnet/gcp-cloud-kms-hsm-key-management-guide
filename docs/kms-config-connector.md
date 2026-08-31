---
title: 5.4 Config Connector
parent: 5. Creating Keys
nav_order: 4
---

# Creating Keys with Config Connector
{: .no_toc }

Kubernetes-native key management. Choose it when your platform is already GitOps
on GKE and you want Google Cloud resources reconciled by the same controller
loop as your workloads.
{: .fs-5 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## When Config Connector beats Terraform

| Situation | Why Config Connector |
|:--|:--|
| GitOps with Argo CD or Flux already in place | Keys become just another manifest in the same sync |
| Continuous reconciliation wanted | The controller re-applies drift automatically, not on a nightly job |
| Application teams self-serve infrastructure | Namespaced resources with RBAC boundaries |
| No state file to secure | State lives in the Kubernetes API and Google Cloud |

| Situation | Why Terraform instead |
|:--|:--|
| No GKE cluster | Config Connector requires one |
| Multi-cloud estate | One tool, one language |
| Complex conditionals and computed values | HCL is more expressive than YAML plus Kustomize |
| Newest KMS features | The CRDs sometimes lag the API |

{: .warning }
> **Continuous reconciliation is a double-edged property for key infrastructure.**
> It is excellent for drift — someone's Console change is reverted within minutes.
> It is dangerous for emergency changes: an IAM binding removed by hand during an
> incident is silently restored by the controller. Know which behavior you want,
> and make sure your incident runbook includes suspending reconciliation.

## Install Config Connector

```bash
# Option A — the GKE add-on (recommended)
gcloud container clusters update platform-prod \
  --location=us-east1 \
  --update-addons=ConfigConnector=ENABLED \
  --workload-pool=contoso-app-prod.svc.id.goog

# The controller's Google service account
gcloud iam service-accounts create cnrm-system \
  --project=contoso-app-prod \
  --display-name="Config Connector controller"

# Grant it what it needs on the KEY project — cloudkms.admin only,
# deliberately NOT any crypto-use role
gcloud projects add-iam-policy-binding contoso-keymgmt-prod \
  --member="serviceAccount:cnrm-system@contoso-app-prod.iam.gserviceaccount.com" \
  --role="roles/cloudkms.admin" --condition=None

# Let the in-cluster KSA impersonate it
gcloud iam service-accounts add-iam-policy-binding \
  "cnrm-system@contoso-app-prod.iam.gserviceaccount.com" \
  --member="serviceAccount:contoso-app-prod.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
  --role="roles/iam.workloadIdentityUser"
```

```yaml
# config-connector/configconnector.yaml
apiVersion: core.cnrm.cloud.google.com/v1beta1
kind: ConfigConnector
metadata:
  name: configconnector.core.cnrm.cloud.google.com
spec:
  mode: cluster
  googleServiceAccount: cnrm-system@contoso-app-prod.iam.gserviceaccount.com
```

```bash
kubectl apply -f config-connector/configconnector.yaml
kubectl wait -n cnrm-system --for=condition=Ready pod --all --timeout=300s
kubectl get pods -n cnrm-system
```

```text
NAME                                      READY   STATUS    RESTARTS   AGE
cnrm-controller-manager-0                 2/2     Running   0          92s
cnrm-deletiondefender-0                   1/1     Running   0          92s
cnrm-resource-stats-recorder-...          2/2     Running   0          94s
cnrm-webhook-manager-...                  1/1     Running   0          94s
```

{: .important }
> **The controller gets `roles/cloudkms.admin` and nothing else.** It must be able
> to create and manage keys; it must never be able to use them. A Config Connector
> controller with `roles/cloudkms.cryptoKeyEncrypterDecrypter` is a cluster-wide
> decryption oracle for anyone who can create a pod in `cnrm-system`.

## The namespace

```yaml
# config-connector/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: keymgmt
  annotations:
    # Every resource in this namespace targets the key-management project
    cnrm.cloud.google.com/project-id: contoso-keymgmt-prod
```

## The key ring

```yaml
# config-connector/keyring.yaml
apiVersion: kms.cnrm.cloud.google.com/v1beta1
kind: KMSKeyRing
metadata:
  name: prod-platform
  namespace: keymgmt
  annotations:
    # Key rings cannot be deleted from GCP. This annotation stops the
    # controller from even trying, and from dropping the resource silently.
    cnrm.cloud.google.com/deletion-policy: "abandon"
  labels:
    environment: prod
    managed-by: config-connector
spec:
  location: us-east1
```

{: .warning }
> **`deletion-policy: abandon` is mandatory for KMS resources.** Without it,
> deleting the manifest makes the controller attempt a delete that the API refuses,
> leaving the resource stuck in a reconcile-error loop and the CR undeletable
> without a finalizer edit. With it, removing the manifest cleanly orphans the
> Google Cloud resource — which is the only outcome the API actually permits.

## The keys

```yaml
# config-connector/keys.yaml
apiVersion: kms.cnrm.cloud.google.com/v1beta1
kind: KMSCryptoKey
metadata:
  name: gcs-cmek-hsm
  namespace: keymgmt
  annotations:
    cnrm.cloud.google.com/deletion-policy: "abandon"
  labels:
    environment: prod
    data-class: confidential
    managed-by: config-connector
spec:
  keyRingRef:
    name: prod-platform
  purpose: ENCRYPT_DECRYPT
  rotationPeriod: 7776000s        # 90 days
  destroyScheduledDuration: 2592000s   # 30 days
  versionTemplate:
    algorithm: GOOGLE_SYMMETRIC_ENCRYPTION
    protectionLevel: HSM
---
apiVersion: kms.cnrm.cloud.google.com/v1beta1
kind: KMSCryptoKey
metadata:
  name: bq-cmek-hsm
  namespace: keymgmt
  annotations:
    cnrm.cloud.google.com/deletion-policy: "abandon"
  labels:
    environment: prod
    data-class: confidential
spec:
  keyRingRef:
    name: prod-platform
  purpose: ENCRYPT_DECRYPT
  rotationPeriod: 7776000s
  destroyScheduledDuration: 2592000s
  versionTemplate:
    algorithm: GOOGLE_SYMMETRIC_ENCRYPTION
    protectionLevel: HSM
---
apiVersion: kms.cnrm.cloud.google.com/v1beta1
kind: KMSCryptoKey
metadata:
  name: artifact-signing-hsm
  namespace: keymgmt
  annotations:
    cnrm.cloud.google.com/deletion-policy: "abandon"
  labels:
    environment: prod
    data-class: internal
spec:
  keyRingRef:
    name: prod-signing
  purpose: ASYMMETRIC_SIGN
  # No rotationPeriod — not supported for asymmetric keys
  destroyScheduledDuration: 2592000s
  versionTemplate:
    algorithm: EC_SIGN_P384_SHA384
    protectionLevel: HSM
```

## IAM bindings

```yaml
# config-connector/iam.yaml
apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMPolicyMember
metadata:
  name: gcs-agent-on-gcs-cmek
  namespace: keymgmt
spec:
  member: serviceAccount:service-123456789012@gs-project-accounts.iam.gserviceaccount.com
  role: roles/cloudkms.cryptoKeyEncrypterDecrypter
  resourceRef:
    apiVersion: kms.cnrm.cloud.google.com/v1beta1
    kind: KMSCryptoKey
    name: gcs-cmek-hsm
---
apiVersion: iam.cnrm.cloud.google.com/v1beta1
kind: IAMPolicyMember
metadata:
  name: key-admins-on-platform-ring
  namespace: keymgmt
spec:
  member: group:gcp-key-admins@contoso.com
  role: roles/cloudkms.admin
  resourceRef:
    apiVersion: kms.cnrm.cloud.google.com/v1beta1
    kind: KMSKeyRing
    name: prod-platform
```

{: .important }
> **Use `IAMPolicyMember`, never `IAMPolicy`.** `IAMPolicy` is authoritative — it
> replaces the whole policy on the resource, silently removing bindings the
> controller does not know about, including service agents Google adds
> automatically. This is the same trap as `google_kms_crypto_key_iam_policy` in
> Terraform, and it produces the same production outage.

## Apply and verify

```bash
kubectl apply -f config-connector/namespace.yaml
kubectl apply -f config-connector/keyring.yaml

kubectl wait --for=condition=Ready kmskeyring/prod-platform \
  -n keymgmt --timeout=300s

kubectl apply -f config-connector/keys.yaml
kubectl apply -f config-connector/iam.yaml

kubectl get kmscryptokey -n keymgmt
```

```text
NAME                   AGE   READY   STATUS     STATUS AGE
artifact-signing-hsm   2m    True    UpToDate   118s
bq-cmek-hsm            2m    True    UpToDate   121s
gcs-cmek-hsm           2m    True    UpToDate   124s
```

```bash
# Full status, including the resolved GCP resource name
kubectl get kmscryptokey gcs-cmek-hsm -n keymgmt -o yaml \
  | yq '.status'
```

```yaml
conditions:
  - lastTransitionTime: "2026-08-18T18:22:31Z"
    message: The resource is up to date
    reason: UpToDate
    status: "True"
    type: Ready
observedGeneration: 1
primary:
  algorithm: GOOGLE_SYMMETRIC_ENCRYPTION
  createTime: "2026-08-18T18:22:29Z"
  name: projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform/cryptoKeys/gcs-cmek-hsm/cryptoKeyVersions/1
  protectionLevel: HSM
  state: ENABLED
selfLink: https://cloudkms.googleapis.com/v1/projects/contoso-keymgmt-prod/locations/us-east1/keyRings/prod-platform/cryptoKeys/gcs-cmek-hsm
```

```bash
# Confirm from the GCP side
gcloud kms keys list --keyring=prod-platform --location=us-east1 \
  --project=contoso-keymgmt-prod \
  --format="table(name.basename(), purpose, versionTemplate.protectionLevel, rotationPeriod)"
```

## Diagnosing a stuck resource

```bash
# The status conditions carry the actual API error
kubectl describe kmscryptokey gcs-cmek-hsm -n keymgmt | sed -n '/Status:/,$p'

# Controller logs
kubectl logs -n cnrm-system -l cnrm.cloud.google.com/component=cnrm-controller-manager \
  --tail=100 | grep -i "kmscryptokey"

# Events
kubectl get events -n keymgmt --sort-by='.lastTimestamp' | tail -20
```

| Symptom | Cause | Fix |
|:--|:--|:--|
| `UpdateFailed`, permission denied | Controller SA lacks `roles/cloudkms.admin` | Grant it on the key project |
| Resource stuck `Deleting` | Missing `deletion-policy: abandon` | Add the annotation, or remove the finalizer |
| `rotationPeriod` rejected | Set on a non-symmetric key | Remove it |
| `alreadyExists` | The key was created out-of-band | Add `cnrm.cloud.google.com/state-into-spec: absent` and let it acquire |
| Drift keeps reverting | That is the intended behavior | Suspend reconciliation during an incident |

```bash
# Suspend reconciliation for one resource during an incident
kubectl annotate kmscryptokey gcs-cmek-hsm -n keymgmt \
  cnrm.cloud.google.com/reconcile-interval-in-seconds="0" --overwrite

# Resume
kubectl annotate kmscryptokey gcs-cmek-hsm -n keymgmt \
  cnrm.cloud.google.com/reconcile-interval-in-seconds- --overwrite
```

{: .tip }
> **Put that suspend command in your incident runbook.** The one situation where
> continuous reconciliation actively hurts is an emergency manual change, and
> discovering the annotation syntax at 3 a.m. while the controller undoes your fix
> every sixty seconds is a bad time to learn it.

## GitOps with Argo CD

```yaml
# config-connector/argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: key-management
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://github.com/steven-smith-itnet/gcp-key-management
    targetRevision: main
    path: config-connector
  destination:
    server: https://kubernetes.default.svc
    namespace: keymgmt
  syncPolicy:
    automated:
      prune: false      # NEVER prune key resources
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

{: .warning }
> **`prune: false` is mandatory for key resources.** With pruning on, removing a
> manifest from Git makes Argo CD delete the CR — which, even with
> `deletion-policy: abandon`, removes your declarative record of a key that still
> exists in Google Cloud. You end up with production keys nothing manages. Prune is
> a good default for stateless workloads and the wrong one for key material.

---

[Next: 5.5 Python client]({% link docs/kms-python.md %}){: .btn .btn-primary }
