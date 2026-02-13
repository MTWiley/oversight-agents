# Reproducibility Reference Checklist

Canonical reference for evaluating whether documentation enables readers to successfully reproduce procedures, set up environments, and execute code examples. Complements the inlined criteria in `review-documentation.md`.

The `documentation-reviewer` agent uses this file as a lookup when evaluating documentation reproducibility. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Installation and Setup Procedures

**Category**: Reproducibility

Installation procedures are the first experience a user has with a project. If the installation steps do not work on first attempt, the project has already lost credibility. Every installation procedure should be copy-pasteable and produce the documented result on the documented platforms.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Installation steps are numbered and sequential | MEDIUM | Unnumbered steps allow readers to skip or reorder inadvertently |
| Each step is a single atomic action, not multiple commands combined | MEDIUM | Multi-action steps make it unclear which sub-step failed |
| Installation verification step is included (a command that confirms success) | HIGH | Without verification, users do not know if installation succeeded |
| Installation steps specify exact commands, not paraphrased instructions | MEDIUM | "Install the dependencies" is not actionable; `pip install -r requirements.txt` is |
| OS-specific differences are documented with clear conditionals | HIGH | A step that works on Linux but fails on macOS with no warning wastes hours |
| Clean-environment installation is documented (not just "works on my machine") | MEDIUM | Steps that depend on pre-existing tools or configurations fail for new users |
| Uninstallation or cleanup procedure is documented | LOW | Users need to undo failed installations or remove the project cleanly |

### Common Mistakes

**Mistake**: Assuming tools are pre-installed without listing them.

Bad:
```markdown
## Installation

1. Clone the repository
2. Run `make build`
3. Run `./bin/server`
```

This assumes Git, Make, a C compiler (if the Makefile compiles C), and potentially other tools are installed. A user on a fresh Ubuntu server or a macOS machine without Xcode command-line tools will fail at step 2.

Good:
```markdown
## Prerequisites

- Git 2.30+
- GNU Make 4.0+
- Go 1.21+ ([install instructions](https://go.dev/doc/install))

Verify prerequisites:
```bash
git --version    # Expected: git version 2.30+
make --version   # Expected: GNU Make 4.0+
go version       # Expected: go1.21+
```

## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/org/project.git
   cd project
   ```

2. Build the binary:
   ```bash
   make build
   ```
   Expected output: `Build complete: bin/server`

3. Verify the build:
   ```bash
   ./bin/server --version
   ```
   Expected output: `server v1.2.3`
```

**Mistake**: Multi-action steps that obscure failure points.

Bad:
```markdown
1. Clone the repo, install dependencies, and build:
   ```bash
   git clone https://github.com/org/project.git && cd project && npm install && npm run build
   ```
```

Good:
```markdown
1. Clone the repository:
   ```bash
   git clone https://github.com/org/project.git
   cd project
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Build the project:
   ```bash
   npm run build
   ```
   Expected output: `Build complete. Output in dist/`
```

**Mistake**: Missing verification step.

Bad:
```markdown
## Installation

1. `pip install mypackage`

You're all set!
```

Good:
```markdown
## Installation

1. Install the package:
   ```bash
   pip install mypackage
   ```

2. Verify the installation:
   ```bash
   python -c "import mypackage; print(mypackage.__version__)"
   ```
   Expected output: `1.2.3`
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Installation procedure with no verification step | HIGH |
| OS-specific step with no platform annotation, causing silent failure on other platforms | HIGH |
| Steps assume undocumented prerequisites | MEDIUM |
| Multi-command steps chained with `&&` without individual explanations | MEDIUM |
| Installation procedure that works and verifies on first attempt | INFO (positive) |

---

## 2. Prerequisite Documentation

**Category**: Reproducibility

Prerequisites are the foundation that installation and usage steps build on. Incomplete prerequisite documentation is the single most common cause of "it doesn't work" reports. Every external dependency -- runtime, tool, service, library, account, or access permission -- must be listed.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| All required runtimes are listed with minimum version numbers | HIGH | Wrong runtime version is the #1 cause of setup failures |
| All required system tools are listed (compilers, build tools, package managers) | MEDIUM | Build failures from missing system tools are confusing for newcomers |
| All required external services are listed (databases, message queues, cloud services) | HIGH | Missing service dependencies cause runtime failures that setup docs do not explain |
| All required accounts or access permissions are listed (cloud accounts, API keys, registry access) | HIGH | Undocumented access requirements block users who do not have privileges |
| Hardware requirements are stated for resource-intensive projects (RAM, disk, GPU) | MEDIUM | Resource-intensive builds that fail with OOM give no indication of the cause |
| Network requirements are documented (internet access for downloads, specific port availability, proxy support) | MEDIUM | Corporate environments with restricted internet access need proxy guidance |
| Version compatibility matrix exists for projects supporting multiple runtime versions | MEDIUM | Users need to know which combinations of runtime versions are supported |
| Prerequisite verification commands are provided | MEDIUM | Users need to confirm prerequisites before starting installation |

### Detection Patterns

**Implicit prerequisites**: Scan the project's build files and scripts for tools and services that must be available but are not documented.

Sources to check:
```
Makefile           -> make, compiler, shell tools referenced in recipes
Dockerfile         -> base image implies tools; build stages may need more
docker-compose.yml -> services listed (postgres, redis, etc.) are prerequisites
package.json       -> Node.js version, npm/yarn/pnpm
pyproject.toml     -> Python version, build tools (setuptools, poetry, hatch)
go.mod             -> Go version
Cargo.toml         -> Rust toolchain version
.tool-versions     -> All runtimes managed by asdf
.github/workflows/ -> Runtimes and tools used in CI (often a superset of what docs mention)
```

**Undocumented services**: Look for connection strings, client initialization, or environment variables in the code that reference external services not mentioned in prerequisites:

```
DATABASE_URL, REDIS_URL, RABBITMQ_URL, KAFKA_BROKERS,
ELASTICSEARCH_HOST, SMTP_HOST, S3_BUCKET, VAULT_ADDR
```

Each of these implies an external service that must be running or accessible.

### Common Mistakes

**Mistake**: Documenting runtime but not build tools.

Bad:
```markdown
## Prerequisites
- Python 3.11+
```

The project's `Makefile` uses `gcc`, `pkg-config`, and `libffi-dev` to compile a C extension. None of these are listed.

Good:
```markdown
## Prerequisites

### Runtime
- Python 3.11+

### Build Tools (required for installation from source)
- GCC 11+ or Clang 14+
- pkg-config
- libffi development headers

#### Debian/Ubuntu
```bash
sudo apt-get install build-essential pkg-config libffi-dev
```

#### macOS
```bash
xcode-select --install
brew install pkg-config libffi
```

#### RHEL/Fedora
```bash
sudo dnf install gcc pkg-config libffi-devel
```
```

**Mistake**: Listing prerequisites without version constraints.

Bad:
```markdown
## Prerequisites
- Node.js
- PostgreSQL
- Redis
```

Good:
```markdown
## Prerequisites
- Node.js 18.0+ (LTS recommended; tested with 18.x and 20.x)
- PostgreSQL 14+ (13 may work but is not tested)
- Redis 7.0+ (required for Streams support used by the job queue)
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| External service required but not listed in prerequisites | HIGH |
| Runtime listed without minimum version number | HIGH |
| Accounts or access permissions required but not documented | HIGH |
| Build tools omitted from prerequisites | MEDIUM |
| Hardware requirements undocumented for resource-intensive project | MEDIUM |
| No prerequisite verification commands provided | MEDIUM |
| Comprehensive prerequisites with versions, verification, and platform-specific install instructions | INFO (positive) |

---

## 3. Step Ordering and Completeness

**Category**: Reproducibility

Procedures must be complete and correctly ordered. A missing step or a step in the wrong order means the procedure fails. Users following documentation expect that executing steps in order, exactly as written, produces the documented result.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Steps can be followed in the documented order without jumping ahead or back | HIGH | Out-of-order steps cause failures that users cannot diagnose |
| No implicit steps exist between documented steps ("now, having configured X..." when X was never documented) | HIGH | Undocumented steps are invisible gaps that cause failure |
| Steps that require waiting (compilation, provisioning, propagation) document the expected wait time | MEDIUM | Users who do not wait may proceed to the next step prematurely |
| Steps that produce side effects (creating files, modifying system state) document those side effects | MEDIUM | Unexpected side effects confuse users and complicate troubleshooting |
| Alternative paths (if-then branches in the procedure) are clearly marked and complete | MEDIUM | Partial alternative paths leave users stranded |
| The final state after all steps are completed is documented | MEDIUM | Users need to know what "done" looks like |
| Dependencies between steps are explicit ("this step requires the output from Step 3") | LOW | Implicit dependencies cause failures when steps are modified or reordered |

### Detection Patterns

**Missing steps**: Walk through the procedure mentally (or literally) and identify points where the documented state does not match what the next step requires.

Common gaps:
```
- File creation steps missing (a config file referenced in Step 5 is never created)
- Directory navigation steps missing (step says "run X" but the working directory changed)
- Service start steps missing (Step 3 needs the database running, but no step starts it)
- Permission steps missing (a step creates a file but a later step needs it executable)
- Network steps missing (a step accesses a URL but firewall rules were never configured)
```

**Ordering violations**: Check that each step's inputs are produced by a preceding step, not a subsequent one.

```
Step 1: Start the application         <- Needs config file from Step 3
Step 2: Run the health check           <- Needs app running from Step 1 (OK)
Step 3: Create the configuration file  <- Should be Step 1
```

**Incomplete alternative paths**: When a procedure branches ("If you are using Docker..." / "If you are installing natively..."), verify that both paths are complete through to the same end state.

Bad:
```markdown
## Setup

If you are using Docker:
```bash
docker compose up -d
```

If you are installing natively:
1. Install Go 1.21+
2. Run `make build`

## Verify Installation
```bash
curl http://localhost:8080/health
```
```

The Docker path is complete (one step to a running service). The native path builds the binary but never starts it. The verification step will fail for native installs because no step runs the server.

Good:
```markdown
## Setup

### Option A: Docker

```bash
docker compose up -d
```

The service starts automatically and is available at `http://localhost:8080`.

### Option B: Native Installation

1. Install Go 1.21+ ([instructions](https://go.dev/doc/install))
2. Build the binary:
   ```bash
   make build
   ```
3. Start the server:
   ```bash
   ./bin/server
   ```
   The service is available at `http://localhost:8080`.

## Verify Installation (both options)

```bash
curl http://localhost:8080/health
```
Expected output: `{"status": "ok"}`
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Missing step causes procedure to fail with no workaround visible in docs | HIGH |
| Steps in wrong order, requiring user to discover correct order through trial and error | HIGH |
| Alternative path documented incompletely (some branches lead to dead ends) | MEDIUM |
| Missing wait time documentation for asynchronous operations | MEDIUM |
| Final state after procedure completion is not documented | MEDIUM |
| Dependencies between steps are implicit but discoverable | LOW |
| Complete, correctly ordered procedure that works on first attempt | INFO (positive) |

---

## 4. Verification Steps

**Category**: Reproducibility

A verification step is a command or check that confirms a procedure succeeded. Without verification steps, users complete a procedure without knowing whether it actually worked. They discover failure later, often in a context far removed from the original procedure, making diagnosis difficult.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Every installation procedure ends with a verification step | HIGH | Users need confirmation that the installation succeeded |
| Every configuration change includes a way to verify the configuration took effect | MEDIUM | Silent configuration failures are the hardest to diagnose |
| Every deployment procedure includes a health check or smoke test | HIGH | Deploying without verification is deploying blind |
| Verification steps show the expected output | MEDIUM | A verification command without expected output is not verifiable |
| Verification steps are specific (check the exact feature, not just "it runs") | MEDIUM | `server --version` verifies the binary exists; it does not verify database connectivity |
| Verification steps can distinguish success from partial success | LOW | A health endpoint returning 200 even when the database is down gives false confidence |
| Negative verification is included where relevant ("you should NOT see...") | LOW | Some verification requires confirming the absence of an error |

### Common Mistakes

**Mistake**: Verification that checks the wrong thing.

Bad:
```markdown
## Verify Installation

Check that the service is installed:
```bash
which myservice
```
```

This verifies the binary is on `$PATH`. It does not verify that the service runs, connects to its dependencies, or serves requests.

Good:
```markdown
## Verify Installation

1. Start the service:
   ```bash
   myservice start
   ```

2. Check the health endpoint:
   ```bash
   curl http://localhost:8080/health
   ```
   Expected output:
   ```json
   {
     "status": "healthy",
     "version": "1.2.3",
     "database": "connected",
     "cache": "connected"
   }
   ```

3. If the health check shows `"database": "disconnected"`, verify your
   `DATABASE_URL` environment variable points to a running PostgreSQL
   instance (see [Database Setup](#database-setup)).
```

**Mistake**: Verification without expected output.

Bad:
```markdown
## Verify

Run the following command:
```bash
myservice check-config
```
```

The reader does not know what success looks like. Does it print "OK"? Does it exit silently? Does it print the parsed configuration?

Good:
```markdown
## Verify Configuration

```bash
myservice check-config
```

Successful output:
```
Configuration valid.
  Database: postgresql://localhost:5432/mydb
  Cache: redis://localhost:6379/0
  Log level: info
  Workers: 4
```

If you see `Configuration error:` followed by an error message, see the
[Configuration Troubleshooting](#configuration-troubleshooting) section.
```

**Mistake**: Only verifying the happy path.

Bad:
```markdown
## Test the API

```bash
curl http://localhost:8080/api/v1/users
```
Expected output: `[{"id": 1, "name": "Admin"}]`
```

This verifies a single endpoint returns data. It does not verify authentication, error handling, or other endpoints.

Good:
```markdown
## Verify the API

1. Test unauthenticated access (should be rejected):
   ```bash
   curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/v1/users
   ```
   Expected output: `401`

2. Test authenticated access:
   ```bash
   curl -H "Authorization: Bearer ${API_TOKEN}" http://localhost:8080/api/v1/users
   ```
   Expected output:
   ```json
   [{"id": 1, "name": "Admin", "role": "admin"}]
   ```

3. Test error handling:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" \
     -H "Authorization: Bearer ${API_TOKEN}" \
     http://localhost:8080/api/v1/users/99999
   ```
   Expected output: `404`
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Installation procedure with no verification step at all | HIGH |
| Deployment procedure with no health check or smoke test | HIGH |
| Verification step without documented expected output | MEDIUM |
| Verification checks binary existence but not functionality | MEDIUM |
| Configuration change with no way to verify it took effect | MEDIUM |
| Verification covers only the happy path | LOW |
| Comprehensive verification with expected outputs, error cases, and troubleshooting links | INFO (positive) |

---

## 5. Platform-Specific Instructions

**Category**: Reproducibility

Software runs on multiple platforms. Documentation that works only on the author's platform silently fails for everyone else. Platform-specific differences must be documented explicitly, not discovered through error messages.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| All supported platforms are listed | MEDIUM | Users need to know if their platform is supported before investing time |
| Platform-specific commands are labeled (Linux, macOS, Windows) | HIGH | An unlabeled command that only works on Linux wastes macOS/Windows users' time |
| Package manager differences are documented (apt, brew, dnf, choco, winget) | MEDIUM | Each OS has different package managers and package names |
| Path conventions are documented or abstracted (`/etc/` vs. `C:\Program Files\`) | MEDIUM | Hard-coded Unix paths fail on Windows; hard-coded Windows paths fail on Unix |
| Shell differences are documented (bash vs. zsh vs. PowerShell) | LOW | Shell syntax differences cause failures (e.g., array syntax, env var expansion) |
| File permission commands are platform-appropriate | LOW | `chmod` does not exist on Windows; NTFS ACLs work differently |
| Architecture-specific differences are documented (x86_64 vs. arm64) | MEDIUM | Increasingly relevant with Apple Silicon and ARM servers |
| Container vs. native differences are documented | MEDIUM | Users running in Docker vs. bare metal may have different networking, paths, and permissions |

### Detection Patterns

**Platform-blind commands**: Look for commands that assume a specific platform without labeling:

```
apt-get install    -> Debian/Ubuntu only
brew install       -> macOS only
yum install        -> RHEL/CentOS 7 only
dnf install        -> RHEL/Fedora 8+
choco install      -> Windows only
pacman -S          -> Arch Linux only
```

**Unix-only path assumptions**:
```
/etc/myapp/config.yaml        -> No Windows equivalent stated
~/.config/myapp/              -> Windows uses %APPDATA% or %LOCALAPPDATA%
/var/log/myapp/               -> Windows uses Event Log or custom paths
/tmp/                         -> Windows uses %TEMP%
```

**Shell-specific syntax**:
```
export VAR=value              -> bash/zsh; PowerShell uses $env:VAR = "value"
source ~/.bashrc              -> bash; zsh uses ~/.zshrc; PowerShell uses $PROFILE
$(command)                    -> bash/zsh; PowerShell uses $(command) but may need different escaping
./script.sh                   -> Unix; Windows needs script.bat or .\script.ps1
```

### Common Mistakes

**Mistake**: Providing only one platform's installation instructions.

Bad:
```markdown
## Install

```bash
brew install myapp
```
```

Good:
```markdown
## Install

### macOS

```bash
brew install myapp
```

### Debian / Ubuntu

```bash
sudo apt-get update
sudo apt-get install myapp
```

### RHEL / Fedora

```bash
sudo dnf install myapp
```

### Windows

```powershell
choco install myapp
```

### From Source (all platforms)

See [Building from Source](./building.md) for instructions.
```

**Mistake**: Using bash-specific syntax without noting the shell requirement.

Bad:
```markdown
## Configuration

Set the environment variables:
```bash
export DATABASE_URL="postgres://localhost:5432/mydb"
export REDIS_URL="redis://localhost:6379"
```
```

Good:
```markdown
## Configuration

Set the environment variables for your shell:

**bash / zsh** (Linux, macOS):
```bash
export DATABASE_URL="postgres://localhost:5432/mydb"
export REDIS_URL="redis://localhost:6379"
```

**PowerShell** (Windows):
```powershell
$env:DATABASE_URL = "postgres://localhost:5432/mydb"
$env:REDIS_URL = "redis://localhost:6379"
```

**fish**:
```fish
set -x DATABASE_URL "postgres://localhost:5432/mydb"
set -x REDIS_URL "redis://localhost:6379"
```

To persist these across sessions, add them to your shell profile or use
a `.env` file (see [Environment Configuration](#environment-configuration)).
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Platform-specific command with no platform label, causing failure on other platforms | HIGH |
| Only one platform's instructions provided when the project supports multiple | MEDIUM |
| Path conventions hard-coded for one OS without alternatives | MEDIUM |
| Architecture differences undocumented (x86_64/arm64) | MEDIUM |
| Shell-specific syntax used without labeling which shell | LOW |
| Comprehensive multi-platform instructions with labeled sections | INFO (positive) |

---

## 6. Configuration Documentation

**Category**: Reproducibility

Configuration is the bridge between software and the user's environment. Every configurable parameter needs documentation that enables users to set the right value for their context without guessing, experimenting, or reading source code.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| A complete example configuration file exists with all parameters and comments | HIGH | Example configs are the most-used reference; incomplete examples leave gaps |
| Every parameter documents: name, type, default, valid range, and description | HIGH | Missing any of these forces users to guess |
| Required vs. optional parameters are clearly distinguished | HIGH | Users need to know the minimum viable configuration |
| Parameter interactions and conflicts are documented ("X and Y are mutually exclusive") | MEDIUM | Conflicting parameters cause confusing runtime errors |
| Configuration file location and format are documented | MEDIUM | Users need to know where to put the file and what format to use |
| Configuration reload behavior is documented (requires restart vs. hot reload) | MEDIUM | Users need to know if they must restart after changes |
| Configuration precedence is documented (env var > config file > default) | MEDIUM | Without precedence rules, users cannot predict which value takes effect |
| Sensitive parameters are identified and documented with security guidance | MEDIUM | Users need to know which values should be in secret stores, not config files |

### Detection Patterns

**Incomplete example configuration**: Compare the parameters in the code's config struct or schema against the example configuration file. Missing parameters are gaps.

**Undocumented interactions**: Look for code that validates parameter combinations:
```python
if config.use_tls and not config.cert_path:
    raise ConfigError("cert_path required when use_tls is True")
```

If this interaction is not documented, users discover it only through the error message.

**Missing precedence documentation**: Look for layered configuration loading:
```python
config = load_defaults()
config.update(load_file("config.yaml"))
config.update(load_env_vars())
config.update(load_cli_args())
```

This establishes precedence: CLI args > env vars > config file > defaults. If this precedence is not documented, users cannot predict behavior when the same parameter is set in multiple locations.

### Common Mistakes

**Mistake**: Example configuration with no comments or descriptions.

Bad:
```yaml
# config.yaml
server:
  port: 8080
  host: 0.0.0.0
  timeout: 30
database:
  url: postgres://localhost:5432/mydb
  max_connections: 10
  idle_timeout: 300
logging:
  level: info
  format: json
```

Good:
```yaml
# config.yaml - Complete configuration reference
# Required parameters are marked with [REQUIRED].
# All other parameters show their default values.

server:
  # [REQUIRED] Port the HTTP server listens on.
  # Type: integer (1024-65535)
  port: 8080

  # Bind address. Use 0.0.0.0 for all interfaces, 127.0.0.1 for local only.
  # Type: string (IP address)
  # Default: "127.0.0.1"
  host: "0.0.0.0"

  # Request timeout in seconds. Requests exceeding this duration are
  # terminated with HTTP 504. Set higher for long-running operations.
  # Type: integer (1-3600)
  # Default: 30
  timeout: 30

database:
  # [REQUIRED] PostgreSQL connection string.
  # Format: postgres://USER:PASSWORD@HOST:PORT/DBNAME?sslmode=MODE
  # SECURITY: Use environment variable DATABASE_URL instead of
  # storing credentials in this file.
  url: "postgres://localhost:5432/mydb"

  # Maximum number of open database connections.
  # Set to ~2x the number of application workers.
  # Type: integer (1-1000)
  # Default: 10
  max_connections: 10

  # Time in seconds before idle connections are closed.
  # Type: integer (0-3600, 0 = never close)
  # Default: 300
  idle_timeout: 300

logging:
  # Log verbosity level.
  # Type: enum (debug, info, warn, error)
  # Default: "info"
  # Note: "debug" produces high-volume output; do not use in production.
  level: "info"

  # Log output format.
  # Type: enum (json, text)
  # Default: "json"
  # Use "json" for production (machine-parseable).
  # Use "text" for local development (human-readable).
  format: "json"
```

**Mistake**: Not documenting which parameters require a restart.

Bad:
```markdown
## Configuration

Edit `config.yaml` to change settings.
```

Good:
```markdown
## Configuration

Edit `config.yaml` to change settings.

### Reload Behavior

| Parameter | Reload Method |
|-----------|--------------|
| `server.port` | Requires restart |
| `server.timeout` | Hot reload (SIGHUP or `POST /admin/reload`) |
| `database.url` | Requires restart |
| `database.max_connections` | Hot reload |
| `logging.level` | Hot reload |
| `logging.format` | Requires restart |

To hot-reload changed parameters without restart:
```bash
kill -SIGHUP $(pidof myservice)
```
Or:
```bash
curl -X POST http://localhost:8080/admin/reload
```
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No example configuration file provided | HIGH |
| Required parameters not distinguished from optional | HIGH |
| Parameters missing type, default, or range information | HIGH |
| Configuration precedence not documented | MEDIUM |
| Reload behavior (restart vs. hot reload) not documented | MEDIUM |
| Parameter interactions not documented | MEDIUM |
| Sensitive parameters not identified with security guidance | MEDIUM |
| Comprehensive configuration reference with annotated example | INFO (positive) |

---

## 7. Environment Variable Documentation

**Category**: Reproducibility

Environment variables are a primary configuration mechanism, especially in containerized and cloud-native environments. Undocumented environment variables are invisible to users. Unlike configuration files that can be browsed, environment variables exist only if someone tells you they exist.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| All environment variables are listed in a single reference location | HIGH | Scattered env var documentation means some are never found |
| Each env var documents: name, purpose, type, default, and example | HIGH | Missing any field leaves users guessing |
| Required vs. optional env vars are distinguished | HIGH | Users need to know the minimum set to launch the application |
| An `.env.example` file exists with all variables and placeholder values | MEDIUM | Example files are the fastest way to set up a new environment |
| Env var naming follows a consistent pattern (`MYAPP_` prefix, UPPER_SNAKE_CASE) | LOW | Consistent naming makes it clear which env vars belong to this application |
| Env var interaction with config file values is documented (which takes precedence) | MEDIUM | Users need to know if the env var or config file wins |
| Sensitive env vars are identified with guidance on secure storage | MEDIUM | Users need to know which values require secret management |
| Env vars with special formatting requirements document the expected format | MEDIUM | Connection strings, JSON values, comma-separated lists need format examples |

### Detection Patterns

**Undocumented env vars**: Search the codebase for environment variable access patterns:

```python
# Python
os.environ["VAR_NAME"]
os.environ.get("VAR_NAME", "default")
os.getenv("VAR_NAME")

# Node.js
process.env.VAR_NAME
process.env["VAR_NAME"]

# Go
os.Getenv("VAR_NAME")
os.LookupEnv("VAR_NAME")

# Rust
std::env::var("VAR_NAME")

# Shell
$VAR_NAME, ${VAR_NAME}
```

Cross-reference every discovered env var against the documentation. Any env var in code but not in docs is a gap.

**Missing `.env.example`**: Check for the existence of `.env.example` or `.env.template` in the repository root. If it does not exist and the project uses env vars, this is a gap.

### Common Mistakes

**Mistake**: Documenting env vars only in scattered inline comments.

Bad:
```python
# In database.py:
db_url = os.environ.get("DATABASE_URL", "postgres://localhost:5432/app")

# In cache.py:
redis_url = os.environ.get("REDIS_URL", "redis://localhost:6379")

# In email.py:
smtp_host = os.environ["SMTP_HOST"]  # Required, no default
```

The only way to discover all env vars is to search every file in the codebase.

Good (centralized documentation):
```markdown
## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | No | `postgres://localhost:5432/app` | PostgreSQL connection string. Format: `postgres://USER:PASS@HOST:PORT/DBNAME` |
| `REDIS_URL` | No | `redis://localhost:6379` | Redis connection URL for caching |
| `SMTP_HOST` | **Yes** | (none) | SMTP server hostname for outbound email |
| `SMTP_PORT` | No | `587` | SMTP server port |
| `SMTP_USER` | **Yes** | (none) | SMTP authentication username |
| `SMTP_PASS` | **Yes** | (none) | SMTP authentication password. **Store in a secret manager.** |
| `LOG_LEVEL` | No | `info` | Logging verbosity. One of: `debug`, `info`, `warn`, `error` |
```

And the accompanying `.env.example`:
```bash
# .env.example - Copy to .env and fill in required values
# Required variables are marked with [REQUIRED]

# Database
DATABASE_URL=postgres://localhost:5432/app

# Cache
REDIS_URL=redis://localhost:6379

# Email [REQUIRED]
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=your-smtp-username
SMTP_PASS=your-smtp-password

# Logging
LOG_LEVEL=info
```

**Mistake**: Env vars with complex format requirements not documented.

Bad:
```markdown
| `ALLOWED_ORIGINS` | No | `*` | CORS allowed origins |
```

Good:
```markdown
| `ALLOWED_ORIGINS` | No | `*` | Comma-separated list of allowed CORS origins. Example: `https://app.example.com,https://staging.example.com`. Use `*` to allow all origins (not recommended for production). |
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Required environment variable not documented anywhere | HIGH |
| Env vars documented only in scattered code comments, no central reference | HIGH |
| Missing `.env.example` file when project uses 5+ env vars | MEDIUM |
| Env var with complex format (JSON, comma-separated) not showing expected format | MEDIUM |
| Precedence between env vars and config file not documented | MEDIUM |
| Sensitive env vars not flagged for secret management | MEDIUM |
| Comprehensive env var table with all fields, plus `.env.example` | INFO (positive) |

---

## 8. Code Example Quality

**Category**: Reproducibility

Code examples in documentation are the most frequently copied content. A code example that does not work when pasted into a file and executed immediately destroys trust in the documentation. Every code example should be complete, runnable, and produce the documented output.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Code examples are complete (all imports, all setup, all teardown) | HIGH | Incomplete examples require users to guess what is missing |
| Code examples are tested (ideally via CI, at minimum manually verified) | HIGH | Untested examples rot faster than tested ones |
| Code examples include expected output | MEDIUM | Without expected output, users cannot verify they ran the example correctly |
| Code examples use the project's actual API, not a stale or invented API | HIGH | Examples calling non-existent functions are worse than no examples |
| Code examples follow the project's coding style | LOW | Examples are implicitly teaching style; inconsistency confuses users |
| Code examples include error handling when demonstrating error-prone operations | MEDIUM | Examples without error handling teach bad practices |
| Code examples use realistic data, not meaningless placeholders | LOW | `foo`, `bar`, `test123` do not help readers understand the domain |
| Snippet examples (not runnable) are clearly labeled as such | MEDIUM | Users who try to run a snippet waste time on missing context |

### Detection Patterns

**Incomplete examples**: Look for code blocks that are missing:
```
- Import statements (function used but not imported)
- Variable declarations (variable referenced but not defined)
- Initialization code (client/connection used but not created)
- Required configuration (API key, endpoint URL)
- Cleanup code (connection close, file close, resource release)
```

**Stale examples**: Compare function signatures, class names, and method names in examples against the actual codebase. Mismatches indicate stale examples.

```
Example calls: client.get_users(filter="active")
Actual API:    client.list_users(status="active")  <- Method renamed, parameter renamed
```

**Non-runnable snippets without labels**: Look for code blocks that cannot be executed as-is because they show only a portion of a larger file. These need a label like "excerpt", "snippet", or context indicating they are part of a larger file.

### Common Mistakes

**Mistake**: Example with missing imports and setup.

Bad:
```markdown
## Usage

```python
result = client.process(data)
print(result.status)
```
```

Where did `client` come from? Where did `data` come from? What imports are needed?

Good:
```markdown
## Usage

```python
from mypackage import Client

# Initialize the client (uses MYAPP_API_KEY environment variable)
client = Client(base_url="https://api.example.com")

# Process a request
data = {"name": "Alice", "action": "transfer", "amount": 100.00}
result = client.process(data)

print(result.status)   # Output: "completed"
print(result.id)       # Output: "txn_abc123" (varies)
```
```

**Mistake**: Example that works only with a specific, undocumented environment.

Bad:
```markdown
## Quick Test

```bash
curl http://localhost:8080/api/users | jq .
```
```

This requires `jq` installed (not documented) and a running server on port 8080 (not started in this procedure).

Good:
```markdown
## Quick Test

Prerequisite: The server must be running (see [Starting the Server](#starting-the-server)). These commands require `curl` and optionally `jq` for formatted output.

```bash
# Without jq (plain JSON):
curl http://localhost:8080/api/users

# With jq (formatted JSON):
curl http://localhost:8080/api/users | jq .
```

Expected output:
```json
[
  {
    "id": 1,
    "name": "Admin",
    "role": "admin"
  }
]
```
```

**Mistake**: Example that silently ignores errors.

Bad:
```python
conn = database.connect(DATABASE_URL)
result = conn.execute("SELECT * FROM users")
users = result.fetchall()
```

Good:
```python
import sys
from mypackage import database, DatabaseError

try:
    conn = database.connect(DATABASE_URL)
except DatabaseError as e:
    print(f"Connection failed: {e}", file=sys.stderr)
    sys.exit(1)

result = conn.execute("SELECT * FROM users")
users = result.fetchall()
print(f"Found {len(users)} users")

conn.close()
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Code example calls non-existent API (stale example) | HIGH |
| Code example missing imports or setup, cannot run as-is (not labeled as snippet) | HIGH |
| Code example uses an outdated API that still works but is deprecated | MEDIUM |
| Code example missing expected output | MEDIUM |
| Code example missing error handling for error-prone operations | MEDIUM |
| Snippet not labeled as snippet (users try to run it and fail) | MEDIUM |
| Code example uses meaningless placeholder data | LOW |
| Complete, runnable, tested examples with expected output | INFO (positive) |

---

## 9. Copy-Paste Friendliness

**Category**: Reproducibility

Users copy commands and code from documentation directly into their terminals and editors. Documentation must be designed for this workflow. Anything that breaks copy-paste -- hidden characters, prompt symbols, line-continuation errors, or mixed content -- wastes user time.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Shell commands do not include the prompt character (`$` or `#` or `>`) | MEDIUM | Pasting `$ npm install` into a terminal runs `$` as a command, which fails or does something unexpected |
| Multi-line commands use correct continuation syntax for the documented shell | MEDIUM | Incorrect line continuations break when pasted |
| Code blocks contain only code, not interleaved prose or comments that break execution | MEDIUM | A code block with `# Now run this:` between commands cannot be pasted as a block |
| File content examples are complete files, not partial snippets mixed with prose | LOW | Users pasting partial files need to know where in the file the content goes |
| Placeholder values are clearly distinguishable and documented | MEDIUM | `your-api-key` is clearly a placeholder; `abc123` might look like the actual value |
| URLs in code blocks are real or clearly example domains (`example.com`) | LOW | Users should not accidentally send requests to real URLs from documentation examples |
| Tab/space formatting in code blocks matches what the runtime expects | MEDIUM | YAML with tabs instead of spaces fails silently; Makefiles with spaces instead of tabs fail loudly |

### Detection Patterns

**Shell prompt characters in code blocks**: Flag code blocks that include shell prompts:

```
$ npm install          <- The $ is not part of the command
# make build           <- The # is not part of the command (or is it a comment?)
> Get-ChildItem        <- The > is not part of the PowerShell command
user@host:~$ ls        <- The entire prompt string is not part of the command
```

If prompts are used for clarity (to distinguish user input from output), use a non-executable block or clearly document the convention:

Good approach -- separate input and output:
```markdown
Run the install command:
```bash
npm install
```

Expected output:
```
added 245 packages in 12.3s
```
```

**Mixed content in code blocks**: Flag code blocks that contain non-executable lines:

Bad:
```markdown
```bash
# First, create the directory
mkdir -p /opt/myapp

# Next, download the binary
Now download the latest release:
curl -L https://releases.example.com/latest -o /opt/myapp/server

# Make it executable
chmod +x /opt/myapp/server
```
```

The line "Now download the latest release:" is prose inside a code block. Pasting this block into a terminal will attempt to execute it as a command.

Good:
```markdown
Create the directory and download the binary:

```bash
mkdir -p /opt/myapp
curl -L https://releases.example.com/latest -o /opt/myapp/server
chmod +x /opt/myapp/server
```
```

**Ambiguous placeholders**: Check that placeholder values are obviously not real values:

Bad:
```bash
export API_KEY=abc123
export DATABASE_URL=postgres://admin:admin@localhost:5432/production
```

A user might think `abc123` is the actual API key for a test environment, or that `admin:admin` is the intended credential.

Good:
```bash
export API_KEY=<your-api-key>          # Get from https://dashboard.example.com/keys
export DATABASE_URL=postgres://<user>:<password>@<host>:5432/<dbname>
```

Or with a clear convention:
```bash
# Replace all values in angle brackets with your actual values
export API_KEY=<YOUR_API_KEY>
export DATABASE_URL=postgres://<DB_USER>:<DB_PASSWORD>@<DB_HOST>:5432/<DB_NAME>
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Shell prompt characters in code blocks that break paste | MEDIUM |
| Prose mixed into code blocks, breaking execution | MEDIUM |
| Placeholder values that look like real values (ambiguous) | MEDIUM |
| YAML examples with tabs (will cause parse errors) | MEDIUM |
| Multi-line command with incorrect continuation syntax | MEDIUM |
| Code blocks designed for clean copy-paste with no prompts or mixed content | INFO (positive) |

---

## 10. Expected Output Documentation

**Category**: Reproducibility

Documenting expected output transforms instructions from "run this and hope" to "run this and verify." Expected output serves three purposes: it confirms success, it helps diagnose partial failures, and it gives users confidence that they are on the right track.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Key commands show expected output (installation, build, health checks, tests) | MEDIUM | Users need to verify success at each step |
| Expected output distinguishes between exact output and variable portions | MEDIUM | Timestamps, IDs, and versions change between runs; users need to know which parts to ignore |
| Error output is documented for common failure scenarios | MEDIUM | Recognizing a known error message is faster than debugging from scratch |
| Output that varies by platform is labeled | LOW | Different platforms may produce slightly different output |
| Long output shows a representative sample, not the full volume | LOW | 500 lines of build output is not useful; the last 5 lines showing success are |
| Exit codes are documented for CLI tools | LOW | Scripts checking `$?` need to know what 0, 1, and other exit codes mean |

### Common Mistakes

**Mistake**: No expected output at all.

Bad:
```markdown
## Build

```bash
make build
```
```

Good:
```markdown
## Build

```bash
make build
```

Expected output (last 3 lines):
```
Compiling src/main.go...
Linking bin/server...
Build complete: bin/server (12.4 MB)
```
```

**Mistake**: Expected output that includes variable elements without marking them.

Bad:
```markdown
Expected output:
```
Server started at 2024-01-15T14:23:45Z on port 8080
Connected to database in 45ms
Ready to accept connections (PID: 28431)
```
```

A user sees different timestamps, connection times, and PIDs. Are these differences expected? Or did something go wrong?

Good:
```markdown
Expected output (timestamps, PID, and durations will vary):
```
Server started at <timestamp> on port 8080
Connected to database in <N>ms
Ready to accept connections (PID: <pid>)
```

Verify that:
- The port matches your configured port (default: 8080)
- The database connection succeeds (no error message)
- The "Ready to accept connections" line appears
```

**Mistake**: Documenting only success output, not failure output.

Bad:
```markdown
Expected output:
```
All 42 tests passed.
```
```

Good:
```markdown
Expected output on success:
```
All 42 tests passed.
```

If you see test failures, common causes include:
- `ConnectionRefused` errors: The test database is not running.
  Fix: `docker compose up -d postgres`
- `TimeoutError` on integration tests: The API server is slow to start.
  Fix: increase `TEST_TIMEOUT` to 30 seconds
- `PermissionError` on file tests: The temp directory is not writable.
  Fix: check permissions on `/tmp` or set `TMPDIR` to a writable directory
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Critical verification step (installation, deployment) with no expected output | MEDIUM |
| Expected output includes variable elements not marked as variable | MEDIUM |
| Common error output not documented (sends users to Google instead of local troubleshooting) | MEDIUM |
| Expected output documented for success and common failure modes | INFO (positive) |

---

## 11. Failure Mode Documentation

**Category**: Reproducibility

Users will encounter errors. Documentation that covers only the happy path abandons users at the point where they most need help. Failure mode documentation -- error messages, their causes, and their resolutions -- is what separates documentation that works from documentation that works only when everything is perfect.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Common error messages are documented with causes and fixes | HIGH | Error-to-fix mapping is the most valuable troubleshooting content |
| Prerequisite failures have specific diagnostic steps | MEDIUM | "Check your prerequisites" is not helpful; specific commands to verify each prerequisite are |
| Network-related failures document proxy, firewall, and DNS considerations | MEDIUM | Enterprise environments have network restrictions that cause opaque errors |
| Permission-related failures document the required permissions | MEDIUM | Permission errors are common and easy to fix once the required permission is known |
| Known issues section exists for longstanding or platform-specific problems | LOW | Known issues save users from debugging problems that are already understood |
| Error messages in the software include enough context to find the docs | MEDIUM | An error message that says "see docs at <URL>" shortcuts the troubleshooting process |
| Timeout-related failures document expected durations and how to increase them | LOW | Users who hit timeouts need to know whether to wait longer or investigate a hang |

### Detection Patterns

**Missing troubleshooting sections**: Check for the presence of any of these:
- A "Troubleshooting" section in the main README or setup guide
- A dedicated `TROUBLESHOOTING.md` file
- Error-to-resolution tables in relevant procedure documentation
- FAQ sections covering common problems

If none exist and the project has more than a trivial setup process, this is a gap.

**Error messages without documentation**: Search the codebase for user-facing error messages and check whether each appears in documentation:
```
raise ValueError("...")
log.error("...")
fmt.Errorf("...")
console.error("...")
sys.exit("Error: ...")
```

Priority: Error messages in setup, configuration, and initialization paths are the most important to document because they affect the first-run experience.

### Common Mistakes

**Mistake**: Generic troubleshooting advice.

Bad:
```markdown
## Troubleshooting

If something goes wrong, check the logs for errors and ensure all
prerequisites are installed correctly.
```

Good:
```markdown
## Troubleshooting

### "Connection refused" when starting the service

**Cause**: The database is not running or is not accessible at the
configured `DATABASE_URL`.

**Fix**:
1. Verify the database is running:
   ```bash
   pg_isready -h localhost -p 5432
   ```
   Expected output: `localhost:5432 - accepting connections`

2. If using Docker:
   ```bash
   docker compose up -d postgres
   ```

3. Verify the `DATABASE_URL` environment variable matches your
   database configuration:
   ```bash
   echo $DATABASE_URL
   ```

### "Permission denied" when writing to /var/log/myapp

**Cause**: The application runs as a non-root user but `/var/log/myapp`
is owned by root.

**Fix**:
```bash
sudo mkdir -p /var/log/myapp
sudo chown $(whoami):$(whoami) /var/log/myapp
```

### "TLS handshake failed" when connecting to external APIs

**Cause**: Corporate proxy or firewall is intercepting HTTPS connections.

**Fix**:
1. If behind a corporate proxy, set the proxy environment variables:
   ```bash
   export HTTP_PROXY=http://proxy.corp.example.com:8080
   export HTTPS_PROXY=http://proxy.corp.example.com:8080
   export NO_PROXY=localhost,127.0.0.1,.corp.example.com
   ```

2. If your organization uses a custom CA certificate:
   ```bash
   export SSL_CERT_FILE=/path/to/corporate-ca-bundle.crt
   ```
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No troubleshooting documentation for a project with non-trivial setup | HIGH |
| Common setup errors undocumented (connection refused, permission denied, etc.) | HIGH |
| Troubleshooting section exists but gives only generic advice | MEDIUM |
| Network-related failure modes undocumented for enterprise-targeted project | MEDIUM |
| Known issues not documented | LOW |
| Comprehensive error-to-resolution mapping with specific diagnostic steps | INFO (positive) |

---

## 12. Rollback Procedures

**Category**: Reproducibility

Every procedure that changes state -- installation, upgrade, migration, deployment, configuration change -- should have a rollback procedure. When a procedure fails partway through or the result is not what was expected, users need a documented path back to the previous working state.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Upgrade procedures include rollback instructions | HIGH | Failed upgrades without rollback paths cause extended outages |
| Database migration documentation includes rollback/downgrade steps | HIGH | Irreversible schema changes without backup/rollback are high risk |
| Deployment documentation includes rollback to previous version | HIGH | Bad deployments need instant reversal |
| Configuration change documentation notes how to revert | MEDIUM | Users need to undo configuration changes that cause problems |
| Installation documentation includes uninstall/cleanup steps | LOW | Failed installations need to be cleaned up before retry |
| Rollback procedures are tested or at least documented as untested | MEDIUM | Documented but untested rollback procedures may not work |
| Point-of-no-return steps are explicitly identified | HIGH | Users need to know which steps cannot be undone |
| Backup steps are documented before destructive operations | HIGH | Backups before migrations or upgrades enable rollback even without built-in rollback support |

### Common Mistakes

**Mistake**: Upgrade documentation with no rollback path.

Bad:
```markdown
## Upgrade from v1 to v2

1. Stop the service
2. Run the database migration: `./migrate up`
3. Deploy v2
4. Start the service
```

Good:
```markdown
## Upgrade from v1 to v2

### Pre-Upgrade

1. Back up the database:
   ```bash
   pg_dump -Fc mydb > backup_v1_$(date +%Y%m%d_%H%M%S).dump
   ```

2. Note the current version:
   ```bash
   myservice --version
   ```
   Expected: `v1.x.x`

### Upgrade

3. Stop the service:
   ```bash
   systemctl stop myservice
   ```

4. Run the database migration:
   ```bash
   ./migrate up
   ```
   **Point of no return**: If the migration completes successfully,
   rolling back requires restoring from the backup created in Step 1.
   The migration is not reversible.

5. Deploy v2:
   ```bash
   cp bin/server-v2 /opt/myservice/server
   ```

6. Start the service:
   ```bash
   systemctl start myservice
   ```

7. Verify the upgrade:
   ```bash
   curl http://localhost:8080/health
   ```
   Confirm the version in the response is `v2.x.x`.

### Rollback

If the upgrade fails at any point:

**If migration has NOT run (failed at or before Step 4)**:
1. Restore the v1 binary (if replaced):
   ```bash
   cp bin/server-v1 /opt/myservice/server
   ```
2. Start the service:
   ```bash
   systemctl start myservice
   ```

**If migration HAS run (failed at Step 5 or later)**:
1. Restore the database from backup:
   ```bash
   pg_restore -d mydb --clean backup_v1_YYYYMMDD_HHMMSS.dump
   ```
2. Restore the v1 binary:
   ```bash
   cp bin/server-v1 /opt/myservice/server
   ```
3. Start the service:
   ```bash
   systemctl start myservice
   ```
4. Verify rollback:
   ```bash
   curl http://localhost:8080/health
   ```
   Confirm version is `v1.x.x`.
```

**Mistake**: Not identifying the point of no return.

Bad:
```markdown
1. Run the migration
2. Deploy the new version
3. Run the data conversion

Note: If you encounter issues, you can roll back.
```

"You can roll back" gives false confidence. Can you roll back after the data conversion? What if the migration altered the schema in a way that v1 cannot read?

Good:
```markdown
1. Run the schema migration
   - **Rollback**: `./migrate down` reverses the schema changes
2. Deploy the new version
   - **Rollback**: Redeploy v1 (`./deploy v1.2.3`)
3. Run the data conversion
   - **POINT OF NO RETURN**: The data conversion is irreversible.
     After this step, rolling back requires restoring from the
     pre-upgrade backup. The v1 binary cannot read the converted data format.
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Upgrade or migration procedure with no rollback instructions | HIGH |
| Destructive operation with no backup step documented | HIGH |
| Point-of-no-return step not identified in a multi-step procedure | HIGH |
| Deployment procedure with no version rollback instructions | HIGH |
| Configuration change documentation with no revert instructions | MEDIUM |
| Rollback documented but explicitly noted as untested | MEDIUM |
| Installation with no uninstall/cleanup steps | LOW |
| Comprehensive rollback procedures with backup, point-of-no-return markers, and tested rollback steps | INFO (positive) |

---

## Quick-Reference Summary

| Category | Check | Severity |
|----------|-------|----------|
| Installation | No verification step | HIGH |
| Installation | OS-specific step without platform label | HIGH |
| Prerequisites | Runtime listed without version number | HIGH |
| Prerequisites | External service required but not listed | HIGH |
| Prerequisites | Account/access requirements not documented | HIGH |
| Step Ordering | Missing step causes procedure failure | HIGH |
| Step Ordering | Steps in wrong order | HIGH |
| Verification | No verification after installation | HIGH |
| Verification | No health check after deployment | HIGH |
| Platform | Platform-specific command without label | HIGH |
| Configuration | No example configuration file | HIGH |
| Configuration | Required vs optional not distinguished | HIGH |
| Configuration | Parameters missing type/default/range | HIGH |
| Environment Vars | Required env var not documented | HIGH |
| Environment Vars | No central env var reference | HIGH |
| Code Examples | Example calls non-existent API | HIGH |
| Code Examples | Example missing imports/setup, not labeled as snippet | HIGH |
| Failure Modes | No troubleshooting docs for non-trivial project | HIGH |
| Rollback | Upgrade/migration with no rollback | HIGH |
| Rollback | Destructive step with no backup | HIGH |
| Rollback | Point-of-no-return not identified | HIGH |
| Verification | Verification without expected output | MEDIUM |
| Platform | Only one platform documented when multiple supported | MEDIUM |
| Configuration | Reload behavior not documented | MEDIUM |
| Environment Vars | Missing `.env.example` | MEDIUM |
| Code Examples | Missing expected output | MEDIUM |
| Code Examples | Snippet not labeled as snippet | MEDIUM |
| Copy-Paste | Prompt characters in code blocks | MEDIUM |
| Copy-Paste | Ambiguous placeholder values | MEDIUM |
| Expected Output | Variable portions not marked | MEDIUM |
| Failure Modes | Only generic troubleshooting advice | MEDIUM |

---

## Review Procedure Summary

When performing reproducibility review:

1. **Audit prerequisites**: Cross-reference build files, dependency manifests, and code against documented prerequisites. Flag every undocumented runtime, tool, service, or account.
2. **Walk through installation steps**: Mentally (or literally) execute each step in order. Identify missing steps, ordering problems, and implicit assumptions.
3. **Check verification steps**: Every procedure that changes state should end with a verification command that shows expected output.
4. **Assess platform coverage**: Identify all supported platforms and verify each has labeled, complete instructions.
5. **Review configuration documentation**: Compare documented parameters against the code's config schema. Check for missing defaults, types, ranges, and descriptions.
6. **Audit environment variables**: Search code for env var access and compare against documentation. Check for a central reference and `.env.example`.
7. **Test code examples**: Verify that code examples include all imports, setup, and teardown. Check that they call current (not stale) APIs.
8. **Check copy-paste friendliness**: Verify code blocks do not include prompt characters, mixed prose, or ambiguous placeholders.
9. **Verify expected output documentation**: Check that key commands show expected output with variable portions marked.
10. **Assess failure mode coverage**: Check for troubleshooting sections, error-to-resolution mapping, and network/permission failure coverage.
11. **Review rollback procedures**: Check that every state-changing procedure has rollback instructions, backup steps, and point-of-no-return markers.
12. **Classify each finding** using the severity tables above, adjusting for audience and project complexity.
