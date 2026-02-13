# Secrets Detection Reference Checklist

Canonical reference for identifying hardcoded secrets, credentials, and sensitive
material in source code and configuration. Complements the inlined criteria in
`review-security.md`.

---

## 1. Hardcoded Passwords and Credentials

### What to Look For

**Variable / key name patterns** (case-insensitive):
```
password, passwd, pwd, pass, secret, credential, auth_token,
api_key, apikey, access_key, private_key, signing_key,
master_key, encryption_key, db_password, db_pass, admin_pass
```

**Assignment patterns** (regex):
```regex
(?i)(password|passwd|pwd|secret|token|api_key|apikey)\s*[:=]\s*["'][^"']{8,}["']
(?i)(password|passwd|pwd|secret)\s*[:=]\s*[^\s;,]{8,}
```

**Configuration file patterns**:
```regex
# YAML / TOML
(?i)(password|secret|token|key):\s*\S+

# Properties files
(?i)(password|secret|token|key)\s*=\s*\S+

# XML
(?i)<(password|secret|token|key)>[^<]+</(password|secret|token|key)>
```

### Common False Positives
- Placeholder values: `CHANGE_ME`, `TODO`, `xxx`, `your-password-here`, `<PASSWORD>`
- Empty strings or null assignments: `password = ""`, `password: null`
- Environment variable references: `password = os.environ["DB_PASS"]`, `${DB_PASSWORD}`
- Test fixtures with obviously fake values: `password = "test123"` in `*_test.*` files
- Documentation examples with clearly dummy values
- Password policy constants: `MIN_PASSWORD_LENGTH = 12`

### Severity Guidance
| Context | Severity |
|---|---|
| Production config with real credential | **CRITICAL** |
| Credential in non-production code path | **HIGH** |
| Default / fallback password in code | **HIGH** |
| Credential in test fixture with real-looking value | **MEDIUM** |
| Credential pattern but clearly placeholder | **INFO** |

---

## 2. API Keys and Tokens

### What to Look For

**Platform-specific patterns** (regex):

| Platform | Pattern | Example Prefix |
|---|---|---|
| AWS Access Key | `AKIA[0-9A-Z]{16}` | `AKIA...` |
| AWS Secret Key | `(?i)aws_secret_access_key\s*[:=]\s*[A-Za-z0-9/+=]{40}` | 40-char base64 |
| AWS Session Token | `(?i)aws_session_token\s*[:=]\s*\S{100,}` | Long base64 string |
| GitHub PAT (classic) | `ghp_[A-Za-z0-9]{36}` | `ghp_...` |
| GitHub PAT (fine-grained) | `github_pat_[A-Za-z0-9_]{82}` | `github_pat_...` |
| GitHub OAuth | `gho_[A-Za-z0-9]{36}` | `gho_...` |
| GitHub App Token | `ghs_[A-Za-z0-9]{36}` | `ghs_...` |
| GitLab PAT | `glpat-[A-Za-z0-9\-]{20}` | `glpat-...` |
| Slack Bot Token | `xoxb-[0-9]{10,}-[0-9]{10,}-[A-Za-z0-9]{24}` | `xoxb-...` |
| Slack Webhook | `https://hooks\.slack\.com/services/T[A-Z0-9]+/B[A-Z0-9]+/[A-Za-z0-9]+` | URL |
| Stripe Secret Key | `sk_live_[A-Za-z0-9]{24,}` | `sk_live_...` |
| Stripe Publishable | `pk_live_[A-Za-z0-9]{24,}` | `pk_live_...` |
| Twilio API Key | `SK[a-f0-9]{32}` | `SK...` |
| SendGrid API Key | `SG\.[A-Za-z0-9_\-]{22}\.[A-Za-z0-9_\-]{43}` | `SG....` |
| Google API Key | `AIza[A-Za-z0-9_\-]{35}` | `AIza...` |
| Heroku API Key | `[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}` | UUID format |
| npm Token | `npm_[A-Za-z0-9]{36}` | `npm_...` |
| PyPI Token | `pypi-[A-Za-z0-9\-]{50,}` | `pypi-...` |
| Datadog API Key | `(?i)dd_api_key\s*[:=]\s*[a-f0-9]{32}` | 32-char hex |
| JWT | `eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_\-]+` | `eyJ...` |
| Bearer Token | `(?i)bearer\s+[A-Za-z0-9\-._~+/]+=*` | `Bearer ...` |
| Generic Hex Token | `(?i)(token|key|secret)\s*[:=]\s*["']?[a-f0-9]{32,}["']?` | 32+ hex chars |

**General token patterns**:
```regex
# Authorization headers with embedded tokens
(?i)authorization\s*[:=]\s*["']?(Bearer|Basic|Token)\s+[A-Za-z0-9\-._~+/]+=*["']?

# Generic high-entropy strings assigned to key/token variables
(?i)(api_key|apikey|access_token|auth_token|secret_key)\s*[:=]\s*["'][A-Za-z0-9+/=\-_]{20,}["']
```

### Common False Positives
- Test/sandbox keys: `sk_test_...`, `pk_test_...` (Stripe), sandbox API keys
- Example keys in documentation: `AKIAIOSFODNN7EXAMPLE`
- Placeholder tokens: `your-api-key-here`, `INSERT_TOKEN`, `<TOKEN>`
- JWT tokens in test fixtures with known payloads
- Publishable / public keys that are intentionally client-side (verify intent)

### Severity Guidance
| Context | Severity |
|---|---|
| Live / production API key or token | **CRITICAL** |
| Cloud provider credential (AWS, GCP, Azure) | **CRITICAL** |
| Payment processor key (Stripe live, etc.) | **CRITICAL** |
| CI/CD token or webhook URL | **HIGH** |
| Test/sandbox key committed to repo | **MEDIUM** |
| Publishable key in client code (by design) | **INFO** |

---

## 3. Connection Strings

### What to Look For

**Database connection strings**:
```regex
# PostgreSQL
postgres(ql)?://[^:]+:[^@]+@[^/]+/\w+

# MySQL
mysql://[^:]+:[^@]+@[^/]+/\w+

# MongoDB
mongodb(\+srv)?://[^:]+:[^@]+@[^/]+

# Redis
redis://:[^@]+@[^/]+

# MSSQL / SQL Server
(?i)server=[^;]+;.*password=[^;]+

# JDBC
jdbc:(mysql|postgresql|oracle|sqlserver)://[^:]+:[^@]+@

# SQLite with sensitive path
sqlite:///.*\.(db|sqlite3?)
```

**Message queue URIs**:
```regex
# RabbitMQ / AMQP
amqps?://[^:]+:[^@]+@[^/]+

# Kafka (with SASL credentials)
(?i)sasl\.(username|password)\s*[:=]\s*\S+
```

**Cache URIs**:
```regex
# Redis with auth
redis://:[^@]+@[^/]+
(?i)redis_password\s*[:=]\s*\S+

# Memcached (usually no auth, but check for tunneled)
```

**LDAP / Active Directory**:
```regex
ldaps?://[^:]+:[^@]+@
(?i)bind_password\s*[:=]\s*\S+
(?i)bind_dn\s*[:=]\s*\S+.*bind_password
```

### Common False Positives
- `localhost` / `127.0.0.1` connections with default dev credentials
- Docker Compose internal service references: `postgres://db:5432/app`
- Connection string templates: `postgres://USER:PASSWORD@HOST:PORT/DB`
- Environment variable interpolation: `postgres://${DB_USER}:${DB_PASS}@...`
- Example URIs in documentation or comments

### Severity Guidance
| Context | Severity |
|---|---|
| Production database connection string with credentials | **CRITICAL** |
| External service URI with embedded credentials | **CRITICAL** |
| Staging/internal connection string with credentials | **HIGH** |
| Local development connection string with default creds | **LOW** |
| Connection string template or env-var reference | **INFO** |

---

## 4. Cloud Provider Credentials

### What to Look For

**AWS**:
```regex
# Access keys
AKIA[0-9A-Z]{16}
ASIA[0-9A-Z]{16}    # Temporary STS credentials

# Secret keys (context-dependent)
(?i)aws_secret_access_key\s*[:=]\s*[A-Za-z0-9/+=]{40}

# Account IDs (not secrets, but useful for context)
(?i)aws_account_id\s*[:=]\s*\d{12}

# ARN with embedded account info
arn:aws:[a-z0-9\-]+:[a-z0-9\-]*:\d{12}:
```

**Files to inspect**: `~/.aws/credentials`, `*.tfvars`, `terraform.tfstate`,
`serverless.yml`, `sam-template.yaml`, `cdk.context.json`

**GCP**:
```regex
# Service account key files (JSON)
"type":\s*"service_account"
"private_key":\s*"-----BEGIN

# Service account email
[a-z0-9\-]+@[a-z0-9\-]+\.iam\.gserviceaccount\.com

# OAuth client secrets
(?i)client_secret\s*[:=]\s*["'][A-Za-z0-9_\-]{24}["']
```

**Files to inspect**: `*-credentials.json`, `*-service-account.json`, `gcloud-*.json`,
`application_default_credentials.json`

**Azure**:
```regex
# Client / tenant IDs with secrets
(?i)client_secret\s*[:=]\s*["'][A-Za-z0-9~._\-]{34,}["']
(?i)tenant_id\s*[:=]\s*[a-f0-9\-]{36}

# Storage account keys
(?i)account_key\s*[:=]\s*[A-Za-z0-9+/=]{88}

# Connection strings
(?i)DefaultEndpointsProtocol=https;AccountName=[^;]+;AccountKey=[A-Za-z0-9+/=]{88}

# SAS tokens
(?i)[?&]sig=[A-Za-z0-9%+/=]{40,}
```

**Files to inspect**: `*.tfvars`, `azuredeploy.parameters.json`, `appsettings.json`,
`local.settings.json`

### Common False Positives
- AWS example key: `AKIAIOSFODNN7EXAMPLE`
- Terraform plan output referencing `(known after apply)`
- Azure ARM template parameters with `secureString` type (values not present)
- GCP project IDs (not secrets by themselves)
- Cloud SDK config pointing to credential helpers, not raw credentials

### Severity Guidance
| Context | Severity |
|---|---|
| Any cloud provider secret key or credential file | **CRITICAL** |
| Temporary/STS credential in code | **CRITICAL** |
| Storage account key or SAS token | **CRITICAL** |
| Cloud account/project/tenant ID without secrets | **LOW** |
| Service account email without key material | **INFO** |

---

## 5. Cryptographic Material

### What to Look For

**Private keys**:
```regex
-----BEGIN (RSA |EC |DSA |OPENSSH |PGP )?PRIVATE KEY( BLOCK)?-----
-----BEGIN ENCRYPTED PRIVATE KEY-----
```

**Certificates** (may contain private key material):
```regex
-----BEGIN CERTIFICATE-----
-----BEGIN PKCS7-----
-----BEGIN PKCS12-----
```

**File extensions to flag**:
```
*.pem, *.key, *.p12, *.pfx, *.jks, *.keystore,
*.p8, *.der, *.cer (if paired with .key)
```

**SSH keys**:
```regex
ssh-(rsa|dsa|ed25519|ecdsa)\s+[A-Za-z0-9+/=]{100,}
```

**GPG/PGP keys**:
```regex
-----BEGIN PGP PRIVATE KEY BLOCK-----
```

**HMAC / symmetric secrets**:
```regex
(?i)(hmac|signing|symmetric)_?(key|secret)\s*[:=]\s*["'][A-Za-z0-9+/=]{16,}["']
```

### Common False Positives
- Public keys and public certificates (no private material)
- Test/self-signed certificates generated at build time
- Certificate fingerprints or serial numbers
- PEM-encoded public keys in trust stores
- Key length constants: `RSA_KEY_SIZE = 4096`

### Severity Guidance
| Context | Severity |
|---|---|
| Private key committed to repository | **CRITICAL** |
| PKCS12/JKS keystore with password nearby | **CRITICAL** |
| Self-signed cert private key in dev config | **HIGH** |
| Public certificate without private key | **INFO** |
| Public key material | **INFO** |

---

## 6. Encoded or Obfuscated Secrets

### What to Look For

**Base64-encoded secrets**:
```regex
# Base64 strings that decode to known secret prefixes
# Look for high-entropy base64 assigned to secret-like variables
(?i)(secret|password|key|token|credential)\s*[:=]\s*["']?[A-Za-z0-9+/]{40,}={0,2}["']?

# Basic auth headers
(?i)authorization\s*[:=]\s*["']?Basic\s+[A-Za-z0-9+/]+=*["']?
```

**Hex-encoded secrets**:
```regex
(?i)(secret|key|token|password)\s*[:=]\s*["']?[0-9a-fA-F]{32,}["']?
```

**URL-encoded credentials**:
```regex
# Credentials embedded in URLs with percent encoding
https?://[^:]+:%[0-9A-Fa-f]{2}[^@]*@
```

**Obfuscation patterns** (deliberate hiding, treat as high suspicion):
```regex
# String reversal, ROT13, XOR patterns
(?i)(reverse|rot13|xor|decode|deobfuscate)\s*\(\s*["'][A-Za-z0-9+/=]+["']

# Byte arrays that reconstruct strings
\[\s*0x[0-9a-fA-F]{2}(\s*,\s*0x[0-9a-fA-F]{2}){15,}\s*\]

# String concatenation to avoid detection
["'][A-Z]{4}["']\s*\+\s*["'][A-Za-z0-9]{4,}["']
```

### Common False Positives
- Base64-encoded non-secret data: images, serialized objects, protobuf
- Hex-encoded hashes used for integrity checks (SHA-256 digests)
- URL encoding in normal path parameters
- Long hex strings that are UUIDs, commit SHAs, or content hashes
- Test data with high entropy but no actual secret value

### Severity Guidance
| Context | Severity |
|---|---|
| Base64-encoded credential that decodes to real secret | **CRITICAL** |
| Deliberately obfuscated secret (XOR, reversed) | **CRITICAL** (also flag as suspicious) |
| High-entropy string in secret-named variable | **HIGH** |
| Base64 string in non-secret context | **INFO** |

---

## 7. Secret-Adjacent Files

### What to Look For

**Environment and dotfiles**:
```
.env, .env.local, .env.production, .env.staging
.env.*.local
.flaskenv
```

**Credential and config files**:
```
credentials, credentials.json, credentials.yml
service-account*.json
*-keyfile.json
.netrc, .npmrc (with _authToken), .pypirc
.docker/config.json (with auths)
.kube/config
.ssh/id_*, .ssh/config
```

**Vault and secret manager configs** (may contain tokens):
```
vault.hcl, vault-config.*
.vault-token
consul-token
```

**Infrastructure files with potential secrets**:
```
terraform.tfstate, *.tfstate.backup
*.tfvars (non-example)
ansible-vault (unencrypted)
docker-compose*.yml (with hardcoded passwords)
k8s secrets (base64-encoded, not encrypted)
```

**Package manager auth**:
```
.npmrc, .yarnrc, .yarnrc.yml (with npmAuthToken)
pip.conf, .pypirc
nuget.config (with apikey)
settings.xml (Maven, with server passwords)
```

**Git-related**:
```
.git-credentials
.gitconfig (with credentials)
```

### Gitignore Verification
When these files are found, verify they are listed in `.gitignore`:
```
.env*
*.tfstate
*.tfvars
*credentials*
*.pem
*.key
*.p12
```

### Common False Positives
- `.env.example` or `.env.template` files with placeholder values
- `terraform.tfstate` in `.gitignore` (only flag if actually committed)
- Empty credential files or those that only reference external stores
- Lock files that mention credential packages but contain no secrets

### Severity Guidance
| Context | Severity |
|---|---|
| `.env` with real credentials committed to repo | **CRITICAL** |
| Terraform state file with secrets committed | **CRITICAL** |
| Credential file committed but values are env-var references | **MEDIUM** |
| Secret-adjacent file present but in `.gitignore` | **INFO** |
| `.env.example` with placeholders | **INFO** |

---

## Review Procedure Summary

When performing secrets detection:

1. **Scan variable names and assignments** against the patterns in sections 1-2.
2. **Scan for platform-specific key formats** using the regex patterns in section 2.
3. **Scan connection strings** for embedded credentials per section 3.
4. **Check for cloud credential files and patterns** per section 4.
5. **Flag cryptographic material** per section 5.
6. **Decode suspicious encoded strings** and check for hidden secrets per section 6.
7. **Audit secret-adjacent files** for accidental commits per section 7.
8. **Verify `.gitignore` coverage** for all detected sensitive file types.
9. **Cross-reference with git history**: secrets removed from HEAD may still exist in prior commits.
10. **Classify each finding** using the severity tables above, adjusting for context.
