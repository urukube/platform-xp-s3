# platform-xp-s3

Crossplane XRD package that provides a self-service S3 bucket golden path for the `urukube` platform. ArgoCD auto-discovers this repo via the `platform-custom-xrds` GitHub topic and deploys it to `crossplane-system` on the orchestrator cluster.

## What gets provisioned

Every `US3Bucket` claim creates four AWS resources in the target account:

| Resource | Configurable |
|---|---|
| S3 Bucket | Name (from claim name), region, account |
| Server-side encryption (AES256) | No — always enabled |
| Versioning | Yes — `spec.parameters.versioning` |
| Public access block (all four flags) | Yes — `spec.parameters.blockPublicAccess` (default: `true`) |
| Bucket policy (IP allowlist) | Auto-created when `blockPublicAccess: false` and `allowedCidrs` is set |

## Parameters

| Field | Required | Default | Description |
|---|---|---|---|
| `spec.parameters.awsAccountId` | Yes | — | 12-digit AWS account ID to create the bucket in |
| `spec.parameters.region` | Yes | — | AWS region (e.g. `us-east-1`) |
| `spec.parameters.versioning` | No | `false` | Enable S3 versioning |
| `spec.parameters.blockPublicAccess` | No | `true` | Block all public access (all four AWS flags). Set to `false` together with `allowedCidrs` to enable IP-restricted public access |
| `spec.parameters.allowedCidrs` | No | — | List of CIDRs granted `s3:GetObject`. Required when `blockPublicAccess` is `false`. Creates a bucket policy automatically |
| `spec.parameters.tags` | No | — | Additional tags as key-value pairs |

The bucket name is taken from `metadata.name` of the claim.

## Example claims

Fully blocked (default):
```yaml
apiVersion: storage.platform.urukube.io/v1alpha1
kind: US3Bucket
metadata:
  name: my-app-assets
  namespace: my-team
spec:
  parameters:
    buId: BU001
    awsAccountId: "123456789012"
    region: us-east-1
    versioning: true
```

IP-allowlisted public access:
```yaml
apiVersion: storage.platform.urukube.io/v1alpha1
kind: US3Bucket
metadata:
  name: my-cdn-origin
  namespace: my-team
spec:
  parameters:
    buId: BU001
    awsAccountId: "123456789012"
    region: us-east-1
    blockPublicAccess: false
    allowedCidrs:
      - "203.0.113.0/24"
      - "198.51.100.42/32"
```

## Cross-account setup

The Composition dynamically creates an `aws.upbound.io/v1beta1 ProviderConfig` per claim that chains the orchestrator's IRSA role into the target account:

```
Orchestrator IRSA role → sts:AssumeRole → arn:aws:iam::<awsAccountId>:role/urukube-crossplane-role
```

Each target AWS account must have a role named `urukube-crossplane-role` with:

1. **Trust policy** — allows the orchestrator's Crossplane IRSA role to assume it:

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::<orchestrator-account-id>:role/<crossplane-irsa-role-name>"
  },
  "Action": "sts:AssumeRole"
}
```

2. **Permissions** — the Upbound S3 provider reads many bucket attributes on every reconcile pass (ACL, CORS, website, logging, replication, etc.), so `s3:*` is required:

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

## Files

| File | Purpose |
|---|---|
| `provider.yaml` | Installs `upbound-provider-aws-s3:v1.21.0`; references the `provider-aws-irsa` DeploymentRuntimeConfig created by `orchestrator-custom-addons` |
| `xrd.yaml` | Defines the `XUS3Bucket` / `US3Bucket` API and parameter schema |
| `composition.yaml` | Maps a claim to the four AWS resources and the per-claim ProviderConfig |
