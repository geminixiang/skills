---
name: gcp-iam-principal-audit
description: Audit what IAM roles a GCP principal or service account has, including Workload Identity impersonation and resource-level permissions. Use when asked to inspect GCP permissions, service account access, bucket/dataset/secret IAM, or who can impersonate a service account.
---

# gcp-iam-principal-audit

Use this skill when the user asks what permissions a GCP principal has, especially a service account such as `name@project.iam.gserviceaccount.com`.

## Principles

- Do **not** brute-force every resource first.
- Prefer **Cloud Asset Inventory** to search IAM bindings.
- For service accounts, always distinguish:
  1. **Roles granted to the service account** on org/folder/project/resources.
  2. **Who can impersonate the service account** via the service account's own IAM policy.
- If the user provides a likely project or resource, query that narrow scope first.
- If the project/resource is unknown, use org/folder Cloud Asset search if available, then fallback to per-visible-project Cloud Asset searches.
- Report scope limitations clearly: results only cover resources visible to the current gcloud account and indexed/supported by Cloud Asset Inventory.

## Inputs to identify

- Principal email, e.g. `foo@bar.iam.gserviceaccount.com` or `user@example.com`.
- Principal member string:
  - Service account: `serviceAccount:foo@bar.iam.gserviceaccount.com`
  - User: `user:alice@example.com`
  - Group: `group:team@example.com`
  - Domain: `domain:example.com`
  - Workload Identity principal/principalSet: use the full `principal://...` or `principalSet://...` string.
- Known project, folder, organization, bucket, dataset, secret, etc., if any.

## Recommended workflow for a service account

### 1. Confirm current gcloud context

```sh
gcloud config list
```

### 2. Confirm the service account exists

If the project is known:

```sh
gcloud iam service-accounts describe SA_EMAIL \
  --project PROJECT_ID \
  --format='table(email,projectId,displayName,uniqueId)'
```

If the project is unknown, infer it from the email domain when possible: `SA_NAME@PROJECT_ID.iam.gserviceaccount.com`.

### 3. Check who can impersonate the service account

This is the service account resource's own IAM policy:

```sh
gcloud iam service-accounts get-iam-policy SA_EMAIL \
  --project PROJECT_ID \
  --flatten='bindings[].members' \
  --format='table(bindings.role,bindings.members)'
```

Look especially for:

- `roles/iam.serviceAccountUser`
- `roles/iam.serviceAccountTokenCreator`
- `roles/iam.workloadIdentityUser`
- `principal://...`
- `principalSet://...`

### 4. Search roles granted to the service account with Cloud Asset

Use project scope first if the likely project is known:

```sh
gcloud asset search-all-iam-policies \
  --scope=projects/PROJECT_ID \
  --query='policy:"serviceAccount\:SA_EMAIL"' \
  --format='table(resource,assetType,policy.bindings.role,policy.bindings.members)' \
  --limit=100
```

Important query syntax notes:

- The colon after `serviceAccount` must be escaped as `\:` inside the quoted query term.
- Example:

```sh
gcloud asset search-all-iam-policies \
  --scope=projects/living-bio \
  --query='policy:"serviceAccount\:designers-github-action@living-bio.iam.gserviceaccount.com"'
```

### 5. If the service account may have access in unknown projects

Try organization scope first:

```sh
gcloud organizations list --format='table(displayName,name)'

gcloud asset search-all-iam-policies \
  --scope=organizations/ORG_ID \
  --query='policy:"serviceAccount\:SA_EMAIL"' \
  --format='table(resource,assetType,policy.bindings.role,policy.bindings.members)' \
  --limit=200
```

If org scope returns incomplete/empty results but project scope is known to work, fallback to one Cloud Asset query per visible project:

```sh
for p in $(gcloud projects list --format='value(projectId)'); do
  echo "== $p =="
  gcloud asset search-all-iam-policies \
    --scope="projects/$p" \
    --query='policy:"serviceAccount\:SA_EMAIL"' \
    --format='table(resource,assetType,policy.bindings.role,policy.bindings.members)' \
    --limit=100 2>/dev/null
 done
```

This is broader, but still avoids enumerating every resource manually.

## Checking Workload Identity provider details

If `roles/iam.workloadIdentityUser` refers to a pool/provider, inspect it:

```sh
gcloud iam workload-identity-pools list \
  --project=PROJECT_ID \
  --location=global \
  --format='table(name,displayName,state)'

# Then list providers in the pool
gcloud iam workload-identity-pools providers list \
  --project=PROJECT_ID \
  --location=global \
  --workload-identity-pool=POOL_ID \
  --format=json
```

For GitHub Actions, check:

- `issuerUri: https://token.actions.githubusercontent.com`
- `attributeCondition`, e.g. `assertion.repository_owner == "livingbio"`
- `attributeMapping`, e.g. repository, ref, actor, subject.

You can also search for a Workload Identity member via Cloud Asset:

```sh
gcloud asset search-all-iam-policies \
  --scope=projects/PROJECT_ID \
  --query='policy:"principalSet\://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/attribute.repository_owner/OWNER"'
```

## Targeted resource verification

If the user mentions a specific resource, directly verify its IAM after Cloud Asset search.

### Cloud Storage bucket

```sh
gcloud storage buckets get-iam-policy gs://BUCKET_NAME --format=json
```

Optional capability smoke test with impersonation:

```sh
gcloud --impersonate-service-account=SA_EMAIL storage ls gs://BUCKET_NAME
```

Do not upload/delete test objects unless the user explicitly approves.

## Final response format

Keep the answer concise and separate:

- **Service account existence**: email, project, display name.
- **Roles granted to the principal**: resource → roles.
- **Impersonation / Workload Identity**: who can use the SA and conditions.
- **Known limitations**: org/project scope visibility, Cloud Asset indexing/support limitations.
- **Verification commands used**: only the important ones, not every failed attempt.

## Common gotchas

- `gcloud asset search-all-iam-policies --scope=organizations/...` may return incomplete/empty results if the current account lacks enough org-level Cloud Asset/IAM visibility. Project-scope can still find results.
- Cloud Asset search returns IAM **bindings**, not a perfect computed list of every effective permission.
- Resource-level IAM exists on many services: GCS buckets, BigQuery datasets/tables, Secret Manager secrets, Artifact Registry repos, KMS keys, Pub/Sub topics/subscriptions, Cloud Run services, etc.
- A service account can have no project-level IAM but still have resource-level IAM, such as bucket-only permissions.
