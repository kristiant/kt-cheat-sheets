# AWS S3

**What it is:** Object storage — you put/get **objects** (files, any size) into **buckets**, addressed by a string **key**. Durable (11 nines), effectively unlimited, pay-per-use.

**Why people use it:** Cheap, durable, infinitely scalable storage with fine-grained access control, lifecycle automation, and event triggers — the default place to keep user uploads, backups, static assets, logs, and data-lake files.

**Typically used for:** User-uploaded files, static website/CDN origin, backups, logs/analytics (data lake), and stage-passing in async pipelines (S3 event → Lambda).

> This sheet is conventions-forward — how to *structure* S3 for production SaaS. CLI/SDK reference is below that. Cross-links: [aws-lambda.md](aws-lambda.md) (S3 triggers), [terraform.md](terraform.md), [aws-secrets-manager.md](aws-secrets-manager.md).

---

## Mental model

There are **no folders.** A key is a flat string; `/` is just a convention that tools render as folders.

```
s3://acme-app-prod-uploads/tenants/42/uploads/9f3c/original.pdf
   └ bucket ────────────┘ └ key (one flat string) ───────────┘
```

Your **prefix design is your security, performance, and ops boundary** — it's where IAM scoping, lifecycle rules, per-tenant deletion, and cost metering hook in. Get it right up front; keys are painful to migrate.

---

## Bucket strategy

- **Naming:** globally unique, DNS-safe, lowercase, no underscores. Pattern: `{org}-{app}-{env}-{purpose}` → `acme-app-prod-uploads`. Append an account-id/random suffix if name-sniping is a concern.
- **One bucket per environment, in separate accounts** — never mix prod/staging keys in one bucket.
- **Split buckets by blast radius / data class, not by feature:** `uploads` (user content), `static` (public via CDN), `exports`, `backups`, `logs`. A bad lifecycle rule or leak is then contained.

```
acme-app-prod-uploads     user content (private)
acme-app-prod-static      public assets (CloudFront origin)
acme-app-prod-exports     generated files
acme-app-prod-logs        access logs / CloudTrail data events
```

---

## Prefix & key conventions

**Multi-tenancy — lead the key with the tenant:**

```
tenants/{tenantId}/uploads/{uuid}/{filename}
```

This single choice unlocks per-tenant IAM (`s3:prefix` conditions), per-tenant lifecycle, one-shot deletion, and cost allocation.

**Keys — don't trust user input, don't make them guessable:**
- Use a `uuid` or content hash, not the raw filename: `uploads/{uuid}/original.pdf`. Keep the original filename in object **metadata**, not the key.
- lowercase, `-` over spaces, no special chars; set the correct `Content-Type` on upload.

**Performance — spread writes across prefixes:**
- S3 scales to **3,500 writes / 5,500 reads per second per prefix partition**, and auto-partitions as load grows (no limit on prefix count). A high-cardinality segment early in the key (tenant id, uuid) avoids hot partitions; a **leading timestamp is an anti-pattern** at high write volume.

**Analytics/logs — date-partition (Hive style):**

```
events/year=2026/month=06/day=17/...   # Athena/Glue read this natively
```

---

## Multi-tenancy & isolation

| Approach | Use when |
|---|---|
| **Shared bucket + `tenants/{id}/` prefix** | default — scales to many tenants; IAM scoped by prefix |
| **Bucket per tenant** | strong isolation needed — but a low default bucket quota (hundreds, raisable to ~1,000) caps tenant count |
| **Per-tenant KMS key** | compliance / **crypto-shredding** (delete the key → tenant's data is unrecoverable) |

Scope a role to one tenant's prefix:

```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::acme-app-prod-uploads/tenants/${aws:PrincipalTag/tenantId}/*"
}
```

---

## Production must-haves

- **Block Public Access = on** (account + bucket). Serve public assets via **CloudFront + OAC** (Origin Access Control), never a public bucket.
- **Versioning on** for important data (recover overwrites/deletes); **Object Lock (WORM)** for compliance/audit immutability.
- **Encryption:** SSE-S3 by default; **SSE-KMS** for sensitive data (per-tenant keys = crypto-shredding).
- **Lifecycle rules per prefix:** transition cold data to IA/Glacier; expire `tmp/`.
- **Presigned URLs** for direct browser upload/download — don't proxy bytes through your server.
- **Access logging / CloudTrail data events** → a separate logs bucket.

```jsonc
// lifecycle: expire scratch files, archive old uploads (Terraform / console)
{ "Rules": [
  { "Filter": { "Prefix": "tmp/" },     "Expiration": { "Days": 1 } },
  { "Filter": { "Prefix": "uploads/" }, "Transitions": [{ "Days": 90, "StorageClass": "GLACIER" }] }
]}
```

---

## CLI

```bash
aws s3 cp file.pdf s3://bucket/key.pdf       # upload
aws s3 cp s3://bucket/key.pdf .              # download
aws s3 sync ./dist s3://bucket/ --delete     # mirror a dir (delete extras)
aws s3 ls s3://bucket/tenants/42/            # list a prefix
aws s3 rm s3://bucket/tmp/ --recursive       # delete a prefix
aws s3 presign s3://bucket/key --expires-in 3600   # temporary signed URL
aws s3api put-bucket-versioning --bucket b --versioning-configuration Status=Enabled
```

---

## SDK v3 (TypeScript)

```bash
npm i @aws-sdk/client-s3 @aws-sdk/s3-request-presigner @aws-sdk/lib-storage
```

```ts
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({ region: "ap-southeast-2" });   // module scope — reuse, see aws-lambda.md

await s3.send(new PutObjectCommand({
  Bucket: "acme-app-prod-uploads",
  Key: `tenants/${tenantId}/uploads/${uuid}/original.pdf`,
  Body: buffer,
  ContentType: "application/pdf",
  Metadata: { originalName: filename },     // keep user filename here, not in the key
}));

const res = await s3.send(new GetObjectCommand({ Bucket, Key }));
const text = await res.Body.transformToString();   // or .transformToByteArray()
```

**Presigned URL — let the browser upload/download directly:**

```ts
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const url = await getSignedUrl(
  s3,
  new PutObjectCommand({ Bucket, Key }),
  { expiresIn: 900 },          // 15 min
);   // hand `url` to the client; it PUTs the file straight to S3
```

**Large files — multipart upload (handles >5GB, retries, parallelism):**

```ts
import { Upload } from "@aws-sdk/lib-storage";

await new Upload({ client: s3, params: { Bucket, Key, Body: bigStream } }).done();
```

---

## Practical recipes

**Mirror a build dir to a bucket (delete removed files):**
```bash
aws s3 sync ./dist s3://acme-app-prod-static/ --delete --cache-control "max-age=31536000"
```

**Delete all of one tenant's data:**
```bash
aws s3 rm s3://acme-app-prod-uploads/tenants/42/ --recursive
```

**Presign a download link (expires in 1h):**
```bash
aws s3 presign s3://bucket/tenants/42/exports/report.csv --expires-in 3600
```

**Bulk delete by prefix in code:**
```ts
import { ListObjectsV2Command, DeleteObjectsCommand } from "@aws-sdk/client-s3";
const { Contents } = await s3.send(new ListObjectsV2Command({ Bucket, Prefix: `tenants/${id}/` }));
if (Contents?.length)
  await s3.send(new DeleteObjectsCommand({ Bucket, Delete: { Objects: Contents.map((o) => ({ Key: o.Key! })) } }));
```

**Empty + delete a bucket (⚠️ destructive, irreversible):**
```bash
aws s3 rm s3://bucket --recursive && aws s3 rb s3://bucket
```

**Provision a hardened bucket (Terraform):**
```hcl
resource "aws_s3_bucket" "uploads" { bucket = "acme-app-prod-uploads" }
resource "aws_s3_bucket_public_access_block" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  block_public_acls = true; block_public_policy = true
  ignore_public_acls = true; restrict_public_buckets = true
}
resource "aws_s3_bucket_versioning" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  versioning_configuration { status = "Enabled" }
}
```

---

## Tips

- Design the key prefix (tenant-first) before you write anything — it's your IAM, lifecycle, and deletion boundary, and migrations are painful.
- Never put user filenames or env in keys; uuid/hash in the key, filename in metadata, env in the bucket name.
- Block Public Access on; serve public content via CloudFront + OAC — a public bucket is almost never the right answer.
- Presign for direct browser transfer; don't stream large files through your API.
- Lifecycle-expire `tmp/` and transition cold data — storage cost creeps silently otherwise.
- Reuse one `S3Client` at module scope (Lambda especially); construct per-request and you pay handshake cost every call.
- Turn on versioning for anything you'd cry about losing; add Object Lock where compliance requires immutability.
