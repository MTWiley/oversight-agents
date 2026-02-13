# Technical Writing Reference Checklist

Canonical reference for evaluating the quality of technical writing in documentation, comments, READMEs, guides, and other prose within a codebase. Complements the inlined criteria in `review-documentation.md`.

The `documentation-reviewer` agent uses this file as a lookup when evaluating documentation quality. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Information Hierarchy

**Category**: Document Structure

A well-structured document guides readers from high-level context to specific details. Poor information hierarchy forces readers to scan the entire document to find what they need, or worse, causes them to miss critical information buried in the wrong section.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Document begins with a purpose statement or summary | MEDIUM | Readers need immediate context to decide if the document is relevant |
| Headings follow a logical hierarchy (H1 -> H2 -> H3, no skipped levels) | LOW | Skipped heading levels break navigation, accessibility, and generated TOCs |
| Most important information appears first within each section | MEDIUM | Readers scan top-down; burying critical details at the bottom causes missed information |
| Prerequisites appear before the procedures that require them | HIGH | Missing prerequisites cause setup failures and wasted time |
| Related information is grouped, not scattered across unrelated sections | MEDIUM | Scattered information forces readers to assemble context from multiple locations |
| Table of contents is present for documents longer than 3 screens | LOW | Long documents without a TOC become navigation dead zones |
| Sections are ordered by reader workflow, not author's mental model | MEDIUM | Author-ordered docs (e.g., alphabetical API lists) ignore the reader's actual task sequence |

### Common Mistakes

**Mistake**: Starting a document with implementation history instead of what the component does.

Why it is a problem: A reader opening the document for the first time needs to know what this thing is and why they should care. The changelog of how it evolved is useful only after they understand the basics.

Bad:
```markdown
# Payment Service

This service was originally written in 2019 as part of the billing migration
project. It was refactored in Q3 2021 to support multi-currency. In 2022,
we added Stripe Connect support for marketplace payouts.

## Overview

The payment service processes credit card transactions...
```

Good:
```markdown
# Payment Service

The payment service processes credit card transactions, manages subscription
billing, and handles marketplace payouts via Stripe Connect. It is the
single entry point for all payment operations in the platform.

## Overview
...

## History

Originally built during the 2019 billing migration. Major milestones:
multi-currency support (Q3 2021), Stripe Connect integration (2022).
```

**Mistake**: Placing warnings or prerequisites after the steps they apply to.

Bad:
```markdown
## Installation

1. Run `make install`
2. Start the service with `./start.sh`

> **Warning**: You must have Go 1.21+ installed before running `make install`.
```

Good:
```markdown
## Prerequisites

- Go 1.21 or later
- Make

## Installation

1. Run `make install`
2. Start the service with `./start.sh`
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Prerequisites documented after the steps that require them | HIGH |
| Document lacks any introductory context or purpose statement | MEDIUM |
| Heading levels are skipped (H1 -> H3, missing H2) | LOW |
| Long document (>1000 words) with no table of contents | LOW |
| Information architecture follows a logical reader workflow | INFO (positive) |

---

## 2. Document Structure

**Category**: Document Structure

Structural elements -- headings, sections, lists, tables, code blocks -- are the scaffolding of a document. Consistent structure reduces cognitive load and makes documents scannable. Inconsistent or missing structure forces linear reading of content that should be skimmable.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Each document has exactly one H1 heading that names the subject | LOW | Multiple H1s or no H1 confuses navigation tools and screen readers |
| Sections are neither too long (>500 words without a subheading) nor too granular (<50 words per section) | LOW | Wall-of-text sections are unreadable; over-fragmented sections disrupt flow |
| Lists are used for 3+ parallel items instead of run-on prose | LOW | Lists are faster to scan than embedded comma-separated items in paragraphs |
| Tables are used for structured data with 2+ attributes per item | LOW | Tabular data presented as prose is hard to compare across items |
| Code blocks use fenced syntax with language identifiers | LOW | Unfenced or untyped code blocks lose syntax highlighting and are harder to copy |
| Admonitions (warnings, notes, tips) use a consistent format | LOW | Mixed admonition styles (blockquotes, bold text, callout syntax) confuse readers |
| Paragraphs are focused on a single idea (3-7 sentences) | LOW | Paragraph breaks signal topic shifts; monolithic paragraphs obscure them |
| Horizontal rules or clear visual breaks separate major sections | INFO | Visual breaks help scanning but are stylistic preference |

### Detection Patterns

**Wall-of-text sections**: Look for sections with more than 500 words and no subheadings, lists, tables, or code blocks. These sections should be broken up with structural elements.

**Over-fragmented documents**: Look for documents where most sections contain fewer than 50 words. This often indicates that the author created a section for every sentence, making the document choppy and hard to read sequentially.

**Inconsistent list styles**: Within a single document, look for mixing of:
- Bulleted lists for some enumerations, numbered prose for others
- Inconsistent punctuation on list items (some with periods, some without)
- Mixing of complete sentences and fragments within the same list

**Unfenced code**: Look for indented code blocks (4-space indent) instead of fenced code blocks. Indented code blocks cannot specify a language for syntax highlighting and are harder to distinguish from regular indented text.

### Severity Guidance

| Context | Severity |
|---------|----------|
| Code blocks missing language identifiers throughout the document | LOW |
| Section exceeding 500 words with no structural breaks | LOW |
| Inconsistent admonition formatting within the same document | LOW |
| Document uses consistent, scannable structure throughout | INFO (positive) |

---

## 3. Clarity and Precision

**Category**: Writing Quality

Technical writing must be unambiguous. Vague language, hedging, and imprecise descriptions cause readers to guess at meaning, leading to misconfiguration, incorrect usage, and support requests.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Statements about system behavior are precise, not vague ("the service retries 3 times" vs. "the service retries") | MEDIUM | Vague behavior descriptions lead to wrong assumptions about system capabilities |
| Quantities, thresholds, and limits are stated numerically | MEDIUM | "Large files may cause issues" is not actionable; "files over 100MB may cause OOM" is |
| Conditional behavior is explicit ("when X, then Y" not "Y might happen") | MEDIUM | Ambiguous conditionals leave readers unsure when behavior applies |
| Negative conditions are stated directly ("does not support X") rather than implied by omission | MEDIUM | Omitting unsupported features causes users to discover limitations by failure |
| Default values are documented for every configurable parameter | HIGH | Undocumented defaults force users to read source code or experiment |
| Error states and their causes are described specifically | MEDIUM | "An error may occur" without context is not actionable |
| Version-specific behavior is labeled with the applicable version range | MEDIUM | Unlabeled version-specific content causes confusion across environments |

### Common Mistakes

**Mistake**: Using weasel words and hedging language.

Weasel words to flag:
```
might, may, could, should (when describing system behavior, not recommendations),
possibly, generally, typically, usually, often, sometimes, in some cases,
it is possible that, there may be, up to (without specifying the range)
```

When these words describe system behavior (not recommendations), they signal that the author is uncertain about the actual behavior. Either investigate and document the precise behavior, or document the uncertainty explicitly with a reason.

Bad:
```markdown
The cache is typically refreshed every few minutes. Large datasets might
cause the service to slow down. The API should return results within a
reasonable time.
```

Good:
```markdown
The cache refreshes every 5 minutes (configurable via `CACHE_TTL_SECONDS`,
default: 300). Datasets exceeding 10,000 rows increase query latency by
approximately 200ms per additional 1,000 rows. The API returns results
within 500ms at the 95th percentile for datasets under 10,000 rows.
```

**Mistake**: Using relative terms without a reference point.

Bad:
```markdown
- Ensure you have a recent version of Node.js installed.
- The response time is fast.
- Set the timeout to a high value for large imports.
```

Good:
```markdown
- Requires Node.js 18.0.0 or later (LTS recommended).
- Median response time: 45ms (p99: 200ms) under standard load.
- Set `IMPORT_TIMEOUT_SECONDS=3600` for imports exceeding 1 million rows.
```

**Mistake**: Documenting what the code does without explaining when or why.

Bad:
```markdown
## retry_count

An integer value that controls retry behavior.
```

Good:
```markdown
## retry_count

Number of times to retry a failed API call before returning an error to the
caller. Each retry uses exponential backoff starting at 1 second.

- **Type**: integer
- **Default**: 3
- **Range**: 0-10
- **When to adjust**: Increase for unreliable network environments. Set to 0
  to disable retries (useful for idempotency-sensitive operations).
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Default values undocumented for configurable parameters | HIGH |
| Vague descriptions of system limits or thresholds (no numbers) | MEDIUM |
| Hedging language describing deterministic system behavior | MEDIUM |
| Relative terms without reference points ("recent", "fast", "large") | MEDIUM |
| Parameter documented with type and range but missing "when to adjust" guidance | LOW |
| Precise, quantified documentation with units and ranges | INFO (positive) |

---

## 4. Sentence Quality

**Category**: Writing Quality

At the sentence level, technical writing should be direct, concise, and scannable. Long sentences, passive voice, and unnecessary complexity slow comprehension and increase the chance of misreading.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Sentences are under 25 words on average | LOW | Long sentences in technical content force re-reading |
| Instructions use imperative mood ("Run the command" not "You should run the command" or "The command should be run") | LOW | Imperative voice is direct and unambiguous for procedures |
| Passive voice is avoided unless the actor is genuinely unknown or irrelevant | LOW | Passive voice obscures who or what performs the action |
| Each sentence conveys one idea | LOW | Compound sentences with multiple technical concepts cause confusion |
| Transition words connect related sentences ("therefore", "however", "as a result") | INFO | Logical connectors clarify relationships but are stylistic |
| Nominalizations are avoided ("perform an installation" -> "install") | LOW | Nominal forms add words without adding meaning |
| Filler phrases are eliminated ("it should be noted that", "in order to", "it is important to") | LOW | Filler phrases delay the useful content and add no information |

### Detection Patterns

**Passive voice patterns** (likely indicates unclear attribution):
```
is/was/were/been/being + past participle

Examples to flag:
- "The configuration is loaded by..."  -> "X loads the configuration..."
- "The file should be created..."      -> "Create the file..."
- "An error will be thrown..."         -> "The parser throws an error..."
- "The request was rejected..."        -> "The server rejected the request..."
```

Exception: Passive voice is acceptable when the actor is genuinely irrelevant or unknown:
```
- "The logs are rotated daily."  (the rotation mechanism is unimportant to the reader)
- "TLS 1.0 was deprecated in 2020."  (historical fact, actor is the industry)
```

**Nominalization patterns** (verb turned into a noun, adding unnecessary words):
```
- "perform an installation"     -> "install"
- "make a modification to"     -> "modify"
- "carry out an investigation"  -> "investigate"
- "give consideration to"      -> "consider"
- "provide an explanation of"  -> "explain"
- "make a determination"       -> "determine"
- "conduct a review of"        -> "review"
- "do a migration of"          -> "migrate"
```

**Filler phrase patterns** (words that add length without meaning):
```
- "It should be noted that..."    -> (delete, start with the actual point)
- "In order to..."                -> "To..."
- "It is important to note..."    -> (delete or start with the actual point)
- "As a matter of fact..."        -> (delete)
- "At the end of the day..."      -> (delete)
- "Due to the fact that..."       -> "Because..."
- "In the event that..."          -> "If..."
- "For the purpose of..."        -> "To..." or "For..."
- "At this point in time..."      -> "Now..."
- "Prior to..."                   -> "Before..."
- "Subsequent to..."              -> "After..."
```

### Examples

Bad (passive, wordy, compound):
```markdown
It should be noted that the configuration file is expected to be placed
in the root directory of the project by the user, and it is important
that the file is validated prior to the application being started, because
if the configuration is found to be invalid, an error will be thrown by
the parser and the application will not be started.
```

Good (active, concise, split):
```markdown
Place the configuration file in the project root directory. Validate it
before starting the application. If the configuration is invalid, the
parser throws an error and the application exits.
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Instructions written in passive voice throughout, obscuring required actions | MEDIUM |
| Average sentence length exceeds 30 words across the document | LOW |
| Pervasive filler phrases adding 20%+ to word count without information | LOW |
| Occasional passive voice or long sentence in otherwise clear prose | INFO |
| Concise, active-voice documentation with clear imperative instructions | INFO (positive) |

---

## 5. Terminology Consistency

**Category**: Terminology

Inconsistent terminology creates ambiguity. When the same concept is called "user", "account", "member", and "customer" across different documents, readers cannot tell whether these are synonyms or distinct concepts with different behaviors.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Each concept has a single canonical term used throughout the documentation | MEDIUM | Multiple terms for the same concept create ambiguity |
| Terminology matches code identifiers (if the code says `tenant`, docs say "tenant" not "organization") | MEDIUM | Mismatched terminology between code and docs forces mental translation |
| Abbreviations are defined on first use in each document | LOW | Undefined abbreviations exclude readers unfamiliar with the jargon |
| A glossary exists for projects with domain-specific terminology (>10 domain terms) | LOW | Complex domains need a central terminology reference |
| Product names, tool names, and protocol names use official capitalization | LOW | Incorrect capitalization ("Github" vs. "GitHub", "javascript" vs. "JavaScript") looks unprofessional |
| Units are consistent (always MB or always MiB, not mixed) | MEDIUM | Mixing MB and MiB (or GB and GiB) creates ambiguity about whether powers of 10 or powers of 2 are intended |
| Boolean configuration is documented consistently ("enable/disable" not sometimes "turn on/turn off" and sometimes "activate/deactivate") | LOW | Consistent verb pairs reduce cognitive load |

### Detection Patterns

**Synonym scanning**: For each key concept in the project, check whether multiple terms are used interchangeably. Common synonym clusters to check:

```
User / Account / Member / Customer / Client / Subscriber
Service / Application / App / Server / System / Component
Deploy / Release / Ship / Push / Publish / Roll out
Config / Configuration / Settings / Options / Preferences / Parameters
Token / Key / Secret / Credential / Auth / API key
Database / DB / Data store / Datastore / Storage / Persistence layer
Error / Exception / Failure / Fault / Issue / Problem
```

For each cluster, determine whether the project intends one canonical term or genuinely distinguishes between them. If the latter, document the distinction explicitly.

**Code-to-docs mismatch detection**: Compare key identifiers in the codebase against the terminology in documentation.

```
Code: `class TenantManager`     Docs: "organization management" -> mismatch
Code: `def create_workspace()`  Docs: "create a new project"   -> mismatch
Code: `CACHE_TTL`               Docs: "cache expiration time"  -> acceptable (TTL defined as "time to live" on first use)
```

### Common Mistakes

**Mistake**: Using multiple terms for the same concept without realizing it.

Bad:
```markdown
## Creating a Workspace

To create a new project, click the "New" button. This will initialize
your workspace with default settings. Each organization can have multiple
projects. Administrators can manage team spaces from the settings page.
```

This paragraph uses four terms -- "workspace", "project", "workspace" again, and "team spaces" -- for what appears to be the same concept. A reader cannot tell if these are synonyms or different features.

Good:
```markdown
## Creating a Workspace

To create a new workspace, click the "New" button. This initializes the
workspace with default settings. Each organization can have multiple
workspaces. Administrators manage workspaces from the organization
settings page.
```

**Mistake**: Using an abbreviation without defining it.

Bad:
```markdown
The service uses mTLS for all inter-service communication. Configure the
CSR with your organization's CA details.
```

Good:
```markdown
The service uses mutual TLS (mTLS) for all inter-service communication.
Configure the Certificate Signing Request (CSR) with your organization's
Certificate Authority (CA) details.
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Code identifiers and documentation terms diverge for key concepts | MEDIUM |
| Same concept called different names within the same document | MEDIUM |
| Units mixed (MB/MiB, ms/seconds) without explicit conversion | MEDIUM |
| Abbreviation used without definition on first use | LOW |
| Product names with incorrect capitalization | LOW |
| Consistent terminology with defined glossary | INFO (positive) |

---

## 6. Formatting Standards

**Category**: Formatting

Consistent formatting makes documents predictable. When readers learn the visual language of a documentation set (code is always in backticks, paths are always monospace, warnings are always in callout blocks), they can parse content faster because the formatting itself carries semantic meaning.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Inline code (commands, paths, parameters, values) uses backtick formatting | LOW | Without backticks, `config.yaml` is indistinguishable from prose |
| Code blocks use fenced syntax (triple backtick) with language identifier | LOW | Fenced blocks enable syntax highlighting and clean copy-paste |
| File paths use monospace formatting consistently | LOW | Paths in regular font blend with surrounding text |
| CLI commands are formatted as code and show the expected shell prompt style | LOW | Unclear where commands begin and end without code formatting |
| Environment variables use `UPPER_SNAKE_CASE` formatting in monospace | LOW | Distinguishes env vars from prose and from code-level variables |
| Links use descriptive text, not bare URLs or "click here" | LOW | Bare URLs are unreadable; "click here" provides no context in link lists |
| Images and diagrams include alt text | LOW | Accessibility requirement; also helps when images fail to load |
| Tables have header rows and consistent alignment | LOW | Headerless tables are ambiguous; inconsistent alignment is hard to scan |
| Line length in source Markdown is reasonable (80-120 characters) for diff readability | INFO | Long lines make Markdown diffs harder to review |

### Detection Patterns

**Unformatted inline code**: Look for patterns that should be in backticks but are not:

```
Patterns to flag:
- File paths: anything ending in a file extension (.yaml, .json, .py, .js, .sh, etc.)
- CLI commands: words that are executable commands (npm, docker, kubectl, git, make, pip)
- Configuration keys: snake_case or kebab-case identifiers in prose context
- Values: true/false/null/none when referring to configuration values
- Environment variables: UPPER_SNAKE_CASE words in prose
- Function/method names: camelCase or snake_case with parentheses
```

**Bare URLs**: URLs that appear without descriptive link text:

Bad:
```markdown
For more information, see https://docs.example.com/api/v2/authentication.
```

Good:
```markdown
For more information, see the [authentication guide](https://docs.example.com/api/v2/authentication).
```

**"Click here" links**: Links with non-descriptive anchor text:

Bad:
```markdown
To configure TLS, [click here](https://docs.example.com/tls-setup).
```

Good:
```markdown
See the [TLS configuration guide](https://docs.example.com/tls-setup).
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Commands or paths in prose without code formatting, causing ambiguity | LOW |
| Code blocks without language identifiers throughout the document | LOW |
| Links using "click here" or bare URLs | LOW |
| Images missing alt text | LOW |
| Consistent formatting throughout with proper code formatting | INFO (positive) |

---

## 7. Audience-Appropriate Writing

**Category**: Writing Quality

Documentation must match its intended audience's knowledge level. Over-explaining fundamentals wastes expert time; assuming expertise alienates beginners. The key is identifying the target audience and writing consistently for that level.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Target audience is identified (explicitly or inferrable from content level) | MEDIUM | Without a defined audience, documents oscillate between beginner and expert levels |
| Prerequisite knowledge is stated at the beginning of the document | MEDIUM | Readers need to self-assess whether the document is appropriate for their level |
| Jargon level matches stated audience | MEDIUM | Expert jargon in a getting-started guide excludes the target audience |
| Explanatory depth matches audience needs (not over-explaining basics to experts, not assuming knowledge beginners lack) | MEDIUM | Mismatched depth frustrates readers at both ends |
| External concepts are linked to references rather than re-explained at length | LOW | Linking to existing explanations avoids both over-explaining and under-explaining |
| Examples match the audience's likely technology stack and use cases | LOW | Irrelevant examples do not help readers apply the information |
| Progressive disclosure is used for complex topics (summary first, details available for those who need them) | LOW | Lets different audiences get value at different depths |

### Common Mistakes

**Mistake**: Mixing audience levels within a single document.

Bad:
```markdown
# Getting Started

## Step 1: Clone the Repository

Git is a distributed version control system. To clone a repository means
to create a local copy. Open your terminal (on macOS, search for "Terminal"
in Spotlight) and run:

```bash
git clone git@github.com:org/project.git
```

## Step 2: Configure the gRPC Interceptors

Add your custom unary interceptor to the dial options. If you need both
client-streaming and server-streaming interceptors, compose them using
`grpc.WithChainStreamInterceptor`. Ensure your context propagation aligns
with the OpenTelemetry trace context specification.
```

Step 1 explains what `git` and a terminal are (beginner level). Step 2 assumes deep familiarity with gRPC interceptor patterns and OpenTelemetry (expert level). The reader who needs Step 1 explained cannot follow Step 2.

Good:
```markdown
# Getting Started

**Audience**: Developers familiar with Git, Go, and basic gRPC concepts.

## Step 1: Clone the Repository

```bash
git clone git@github.com:org/project.git
cd project
```

## Step 2: Configure the gRPC Interceptors

Add your custom unary interceptor to the dial options:

```go
conn, err := grpc.Dial(target,
    grpc.WithUnaryInterceptor(loggingInterceptor),
    grpc.WithChainStreamInterceptor(metricsInterceptor, tracingInterceptor),
)
```

For background on gRPC interceptors, see the
[gRPC Go documentation](https://grpc.io/docs/languages/go/interceptors/).
```

**Mistake**: Assuming the reader understands project-internal conventions without explanation.

Bad:
```markdown
Use the standard service pattern to add your handler. Register it in
the bootstrap phase and add the feature flag.
```

Good:
```markdown
1. Create a handler struct implementing the `ServiceHandler` interface
   (see `internal/handler/example_handler.go` for a template).
2. Register the handler in `cmd/server/bootstrap.go` by adding it to
   the `handlers` slice.
3. Add a feature flag in `config/flags.yaml` to gate the new endpoint
   (see [Feature Flags Guide](./feature-flags.md) for naming conventions).
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Document mixes beginner and expert levels, making it unsuitable for either audience | MEDIUM |
| Getting-started guide assumes knowledge that beginners are unlikely to have | MEDIUM |
| No stated or inferrable target audience for the document | MEDIUM |
| Project-internal conventions referenced without explanation or links | MEDIUM |
| Complex concept explained at appropriate depth with links for further reading | INFO (positive) |

---

## 8. Grammar and Style Rules

**Category**: Writing Quality

Grammar errors in documentation erode reader confidence. If the documentation has typos and grammatical mistakes, readers question whether the technical content is equally careless. This section covers the most impactful grammar and style rules for technical writing.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Subject-verb agreement is correct throughout | LOW | Basic grammatical correctness |
| Tense is consistent within each section (present for current behavior, past for changelogs) | LOW | Tense shifts within a section confuse the temporal frame |
| Lists use parallel grammatical structure (all imperative, all noun phrases, or all sentences) | LOW | Non-parallel lists are harder to scan and parse |
| Spelling is correct for technical terms and common words | LOW | Typos erode credibility and may confuse search |
| Pronouns have clear antecedents ("it", "this", "they" refer unambiguously to a specific noun) | MEDIUM | Ambiguous pronouns in technical writing cause misinterpretation |
| Oxford comma is used consistently (either always or never, matching project style) | INFO | Inconsistency is the issue, not the choice itself |
| Numbers follow a consistent style (spell out one through nine, numerals for 10+, always numerals for technical values) | INFO | Consistency aids scanning |
| Contractions are used consistently (either all informal or all formal) | INFO | Mixing formal and informal tone feels inconsistent |

### Detection Patterns

**Non-parallel lists**: Check that all items in a list use the same grammatical structure.

Bad (mixed structures):
```markdown
The service provides:
- Automatic retry of failed requests
- Rate limiting
- You can monitor throughput with the dashboard
- To configure timeouts, edit the config file
```

Good (parallel noun phrases):
```markdown
The service provides:
- Automatic retry of failed requests
- Rate limiting for outbound calls
- Throughput monitoring via the dashboard
- Configurable timeouts via the config file
```

Good (parallel imperative sentences):
```markdown
To configure the service:
- Enable automatic retries by setting `retry_enabled: true`
- Set rate limits with the `rate_limit_rps` parameter
- Monitor throughput using the Grafana dashboard
- Configure timeouts in `config.yaml`
```

**Ambiguous pronouns**: Flag "it", "this", "that", "these", "they" when the antecedent is not the immediately preceding noun or is ambiguous between two possible referents.

Bad:
```markdown
The service sends the request to the gateway. It validates the token and
forwards it to the backend. If it fails, it retries.
```

Questions a reader might ask: Which "it" validates -- the service or the gateway? Which "it" is forwarded -- the request or the token? Which "it" fails -- the validation, the forwarding, the service, or the gateway? Which "it" retries?

Good:
```markdown
The service sends the request to the gateway. The gateway validates the
token and forwards the request to the backend. If validation fails, the
gateway returns a 401 error. If the backend is unreachable, the service
retries the request.
```

**Tense inconsistency**: Within a section describing current behavior, mixing present and future tense.

Bad:
```markdown
The scheduler runs every 5 minutes. It will check for pending tasks and
processes them. If a task fails, it was going to be retried after a delay.
```

Good:
```markdown
The scheduler runs every 5 minutes. It checks for pending tasks and
processes them. If a task fails, the scheduler retries it after a
30-second delay.
```

### Style Rules for Technical Docs

**Sentence style for headings**: Use sentence case ("Getting started with deployment") not title case ("Getting Started With Deployment") unless the project has an established title-case convention. Be consistent.

**Action-oriented headings**: Section headings should tell the reader what the section helps them do, not just what it is about.

Bad:
```markdown
## Configuration
## Database
## Errors
```

Good:
```markdown
## Configure the service
## Connect to the database
## Handle errors
```

**Second person for instructions**: Use "you" for instructions, not "we" or "the user."

Bad:
```markdown
We need to configure the database connection before starting the service.
The user should then verify the connection.
```

Good:
```markdown
Configure the database connection before starting the service. Then
verify the connection.
```

Or, when appropriate:
```markdown
You must configure the database connection before starting the service.
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Ambiguous pronouns in critical procedural documentation (could cause wrong action) | MEDIUM |
| Non-parallel list structure making items difficult to parse | LOW |
| Pervasive tense inconsistency throughout the document | LOW |
| Spelling errors in technical terms (may confuse search and grep) | LOW |
| Occasional minor grammatical issue in otherwise clean prose | INFO |
| Consistently clean grammar with parallel structure and clear antecedents | INFO (positive) |

---

## 9. Completeness of API and Parameter Documentation

**Category**: Completeness

For libraries, services, and tools, the documentation of public interfaces (APIs, CLI arguments, configuration parameters, environment variables) is the most critical documentation. Incomplete parameter documentation forces users to read source code, which defeats the purpose of documentation.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Every public API endpoint is documented with method, path, request body, response body, and status codes | HIGH | Undocumented endpoints are unusable without reading source |
| Every configuration parameter is documented with name, type, default, valid range, and description | HIGH | Missing parameter docs force users to experiment or read code |
| Every CLI command and flag is documented with usage, description, and examples | HIGH | Undocumented CLI flags are invisible to users |
| Every environment variable is documented with name, purpose, default, and example value | MEDIUM | Undocumented env vars are discoverable only by reading code |
| Error codes and error messages are documented with causes and resolutions | MEDIUM | Undocumented errors send users to search engines or support channels |
| Authentication and authorization requirements are documented per endpoint | HIGH | Missing auth docs cause confusing 401/403 errors |
| Rate limits, quotas, and usage constraints are documented | MEDIUM | Undocumented limits surprise users in production |

### Detection Patterns

**Undocumented parameters**: Compare the set of parameters accepted by the code (CLI flags, config file keys, env vars, API fields) against the documentation. Any parameter in code but not in docs is a gap.

**Missing defaults**: Check that every parameter with a default value documents that default. Common pattern:
```
Good:  `timeout` (integer, default: 30) - Request timeout in seconds.
Bad:   `timeout` (integer) - Request timeout in seconds.
```

**Missing type information**: Parameters documented without their type:
```
Good:  `max_connections` (integer, 1-100, default: 10)
Bad:   `max_connections` - Maximum number of connections.
```

**Missing error documentation**: Look for error codes or error messages in the source code that do not appear in documentation. Priority areas:
- HTTP status codes returned by API endpoints
- Exit codes from CLI tools
- Error messages from configuration validation
- Exceptions that callers should handle

### Severity Guidance

| Context | Severity |
|---------|----------|
| Public API endpoints with no documentation | HIGH |
| Configuration parameters missing from documentation | HIGH |
| CLI commands or flags undocumented | HIGH |
| Environment variables undocumented | MEDIUM |
| Parameters documented but missing type, default, or range | MEDIUM |
| Error codes undocumented | MEDIUM |
| Comprehensive parameter documentation with types, defaults, ranges, and examples | INFO (positive) |

---

## 10. Code Comments and Inline Documentation

**Category**: Code Documentation

Comments in source code serve a different purpose than external documentation. They explain why code makes specific choices, document non-obvious constraints, and provide context that cannot be inferred from the code itself. Comments should not explain what the code does (the code should be readable enough for that) but why it does it that way.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Public functions and classes have docstrings or doc comments | MEDIUM | Public interfaces need documentation at the declaration site |
| Comments explain "why", not "what" | LOW | Comments restating the code add noise without value |
| Comments are accurate and up-to-date with the code they describe | MEDIUM | Stale comments are worse than no comments because they mislead |
| TODO/FIXME/HACK comments include context (author, issue number, or explanation) | LOW | Contextless TODOs become permanent noise |
| Complex algorithms or non-obvious business rules have explanatory comments | MEDIUM | Complex logic without rationale forces future readers to reverse-engineer intent |
| Workarounds for bugs or limitations reference the issue number or upstream fix version | MEDIUM | Without a reference, the workaround may remain forever after the bug is fixed |
| Module-level comments describe the module's purpose and responsibilities | LOW | Helps developers understand module boundaries without reading all code |

### Common Mistakes

**Mistake**: Comments that restate the code.

Bad:
```python
# Increment counter by 1
counter += 1

# Check if user is admin
if user.role == "admin":

# Return the result
return result
```

Good:
```python
# Rate limiter uses a token bucket that refills every window reset.
# We increment here rather than in the request handler to avoid
# double-counting when retries hit the same endpoint.
counter += 1

# Admin users bypass the approval workflow per SOX compliance
# requirement SOX-2024-07. Regular users go through the
# standard multi-step approval in approve_request().
if user.role == "admin":
```

**Mistake**: Stale comments that contradict the code.

Bad:
```python
# Default timeout is 30 seconds
DEFAULT_TIMEOUT = 60  # Changed to 60 but comment not updated
```

Good:
```python
# Timeout increased from 30s to 60s to accommodate slow responses
# from the upstream payment provider (see incident INC-2024-0312).
DEFAULT_TIMEOUT = 60
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Public function or class missing documentation entirely | MEDIUM |
| Comment contradicts the actual code behavior | MEDIUM |
| Workaround without reference to the upstream issue | MEDIUM |
| Complex business logic with no explanatory comment | MEDIUM |
| Comment restating obvious code behavior | LOW |
| TODO without context (no ticket, no author, no explanation) | LOW |
| Well-commented code explaining rationale and constraints | INFO (positive) |

---

## Quick-Reference Summary

| Category | Check | Severity |
|----------|-------|----------|
| Information Hierarchy | Prerequisites after steps that need them | HIGH |
| Information Hierarchy | No purpose statement or introduction | MEDIUM |
| Clarity & Precision | Undocumented default values | HIGH |
| Clarity & Precision | Vague thresholds or limits (no numbers) | MEDIUM |
| Clarity & Precision | Hedging language for deterministic behavior | MEDIUM |
| Completeness | Undocumented API endpoints | HIGH |
| Completeness | Undocumented configuration parameters | HIGH |
| Completeness | Undocumented CLI commands or flags | HIGH |
| Completeness | Missing auth requirements per endpoint | HIGH |
| Completeness | Undocumented environment variables | MEDIUM |
| Completeness | Undocumented error codes | MEDIUM |
| Terminology | Code-docs terminology mismatch | MEDIUM |
| Terminology | Same concept, multiple names in one document | MEDIUM |
| Terminology | Mixed units (MB/MiB) | MEDIUM |
| Audience | Mixed beginner/expert levels in one document | MEDIUM |
| Audience | Getting-started assumes unstated knowledge | MEDIUM |
| Code Comments | Stale comment contradicting code | MEDIUM |
| Code Comments | Complex logic with no rationale comment | MEDIUM |
| Code Comments | Public function with no docstring | MEDIUM |
| Sentence Quality | Passive voice throughout procedures | MEDIUM |
| Document Structure | Heading levels skipped | LOW |
| Document Structure | Code blocks without language identifiers | LOW |
| Formatting | Inline code without backtick formatting | LOW |
| Formatting | Bare URLs or "click here" links | LOW |
| Grammar | Non-parallel list items | LOW |
| Grammar | Ambiguous pronouns in procedures | MEDIUM |
| Grammar | Tense inconsistency | LOW |

---

## Review Procedure Summary

When performing technical writing review:

1. **Assess document structure**: Check heading hierarchy, section ordering, presence of introduction and TOC.
2. **Verify completeness**: Cross-reference documented parameters, APIs, and CLI flags against the codebase.
3. **Check clarity and precision**: Flag vague language, missing defaults, undocumented thresholds, and hedging.
4. **Review terminology**: Identify synonym clusters and verify consistency with code identifiers.
5. **Evaluate sentence quality**: Check for passive voice, nominalizations, filler phrases, and sentence length.
6. **Verify formatting**: Check backtick usage for code, language identifiers on code blocks, link quality.
7. **Assess audience fit**: Verify consistent knowledge-level assumptions throughout each document.
8. **Check grammar fundamentals**: Parallel lists, clear pronoun antecedents, tense consistency.
9. **Review code comments**: Check for stale comments, missing rationale, undocumented public interfaces.
10. **Classify each finding** using the severity tables above, adjusting for context and audience impact.
