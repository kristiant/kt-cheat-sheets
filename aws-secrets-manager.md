# AWS Secrets Manager

**What it is:** An AWS service that stores, encrypts, and retrieves secrets (API keys, DB credentials, tokens) via API/IAM, with built-in rotation.

**Why people use it:** Keeps secrets out of code, env files, and git; centralises access control (IAM) and audit (CloudTrail); rotates credentials automatically.

**Typically used for:** Storing DB passwords, third-party API keys (OpenAI/Anthropic/Cohere), and any credential a service fetches at runtime instead of baking in.

> Secrets Manager vs SSM Parameter Store: Parameter Store is cheaper/simpler for plain config; Secrets Manager adds rotation, cross-account, and is purpose-built for credentials.

---

## CLI

```bash
# Create / update
aws secretsmanager create-secret --name prod/openai --secret-string '{"apiKey":"sk-..."}'
aws secretsmanager put-secret-value --secret-id prod/openai --secret-string '{"apiKey":"sk-new"}'

# Read
aws secretsmanager get-secret-value --secret-id prod/openai --query SecretString --output text

# List / delete
aws secretsmanager list-secrets
aws secretsmanager delete-secret --secret-id prod/openai --recovery-window-in-days 7
```

---

## Fetch a secret (Node, SDK v3)

```bash
npm i @aws-sdk/client-secrets-manager
```

```ts
import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";

const client = new SecretsManagerClient({ region: "ap-southeast-2" });

async function getSecret(id: string): Promise<Record<string, string>> {
  const res = await client.send(new GetSecretValueCommand({ SecretId: id }));
  return JSON.parse(res.SecretString!);
}

const { apiKey } = await getSecret("prod/openai");
```

---

## Advanced

### Cache it — don't fetch per request

Secrets Manager calls cost money and add latency. Fetch once at startup (or cache with a TTL); secrets change rarely.

```ts
let cache: { value: Record<string, string>; expires: number } | null = null;

async function getCachedSecret(id: string, ttlMs = 300_000) {
  if (cache && cache.expires > Date.now()) return cache.value;
  const value = await getSecret(id);
  cache = { value, expires: Date.now() + ttlMs };
  return value;
}
```

> **Grounded:** AWS recommends caching rather than calling `GetSecretValue` on every invocation; they ship the [AWS Secrets Manager Caching client](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_cache.html) (and a Lambda extension) for exactly this. Caching also smooths past per-account API rate limits.

### Initialise an LLM client from a secret

```ts
import { ChatOpenAI } from "@langchain/openai";

const { apiKey } = await getCachedSecret("prod/openai");
const llm = new ChatOpenAI({ apiKey, model: "gpt-4o" });
```

### Rotation

```bash
aws secretsmanager rotate-secret \
  --secret-id prod/db \
  --rotation-lambda-arn arn:aws:lambda:...:rotator \
  --rotation-rules AutomaticallyAfterDays=30
```

A rotation Lambda swaps the credential and updates the secret; clients that re-fetch (or use the cache with a TTL) pick up the new value automatically.

### IAM — least privilege

Grant read of only the secrets a service needs:

```json
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:ap-southeast-2:123456789012:secret:prod/openai-*"
}
```

The trailing `-*` covers the random 6-char suffix AWS appends to secret ARNs.

### Terraform

```hcl
resource "aws_secretsmanager_secret" "openai" {
  name = "prod/openai"
}

resource "aws_secretsmanager_secret_version" "openai" {
  secret_id     = aws_secretsmanager_secret.openai.id
  secret_string = jsonencode({ apiKey = var.openai_api_key })
}
```

> Pass `var.openai_api_key` via a `TF_VAR_` env var or `-var`, never hardcode it. See [terraform.md](terraform.md).

---

## Practical recipes

**Read one JSON key from a secret (CLI + jq):**
```bash
aws secretsmanager get-secret-value --secret-id prod/openai \
  --query SecretString --output text | jq -r .apiKey
```

**Update a single key without clobbering the rest:**
```bash
current=$(aws secretsmanager get-secret-value --secret-id prod/app --query SecretString --output text)
updated=$(echo "$current" | jq '.apiKey="sk-new"')
aws secretsmanager put-secret-value --secret-id prod/app --secret-string "$updated"
```

**Load all secret keys into env at boot (Node):**
```ts
const secret = await getCachedSecret("prod/app");
Object.assign(process.env, secret);
```

**Force-delete immediately (no recovery window — ⚠️ irreversible):**
```bash
aws secretsmanager delete-secret --secret-id test/tmp --force-delete-without-recovery
```

---

## Tips

- Store related keys as **one JSON secret** (`{"apiKey":...,"orgId":...}`) rather than many secrets — fewer API calls, one IAM grant.
- Cache fetched secrets (TTL of minutes); don't call `GetSecretValue` per request.
- Scope IAM to specific secret ARNs with the `-*` suffix; never `Resource: "*"`.
- Never log a secret value or put it in an exception message.
- Pick your region deliberately — secrets are regional; cross-region needs replication.
- For plain non-sensitive config, SSM Parameter Store is cheaper.
