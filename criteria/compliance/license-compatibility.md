# License Compatibility Reference Checklist

Canonical reference for evaluating dependency license compliance, license
compatibility, obligation tracking, and automated enforcement. Covers the full
spectrum from permissive to proprietary licenses, compatibility matrices,
vendored code governance, and CI-integrated license scanning.

Complements the inlined criteria in `review-compliance.md`. The
`compliance-reviewer` agent uses this file as a lookup when evaluating license
posture.

---

## 1. Project License Declaration

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 1.1 | A `LICENSE` or `LICENCE` file exists at the repository root | **HIGH** |
| 1.2 | The license file contains the full license text, not just a name or abbreviation | **MEDIUM** |
| 1.3 | The license identifier in the package manifest (`license` field in `package.json`, `pyproject.toml`, `Cargo.toml`, `pom.xml`, etc.) matches the LICENSE file | **MEDIUM** |
| 1.4 | The license uses a valid SPDX identifier (e.g., `MIT`, `Apache-2.0`, not `MIT License` or `Apache 2`) | **LOW** |
| 1.5 | For dual-licensed projects, the SPDX expression uses proper syntax (`MIT OR Apache-2.0`, not `MIT/Apache-2.0`) | **LOW** |
| 1.6 | The license is appropriate for the distribution model: proprietary projects do not use copyleft; open-source projects use an OSI-approved license | **HIGH** |
| 1.7 | Copyright year and holder are specified in the LICENSE file | **LOW** |
| 1.8 | If the project is not open-source, a clear proprietary license or "All Rights Reserved" notice is present | **MEDIUM** |
| 1.9 | Sub-packages in a monorepo each have their own LICENSE file if they are published independently | **MEDIUM** |
| 1.10 | The README or contributing guide states the project license prominently | **LOW** |

### What to Look For

```regex
# License file existence
(?i)^(LICENSE|LICENCE|COPYING|UNLICENSE)(\.md|\.txt)?$

# SPDX identifiers in manifests
(?i)"license"\s*:\s*"([^"]+)"
(?i)license\s*=\s*"([^"]+)"

# Invalid SPDX patterns (common mistakes)
(?i)"license"\s*:\s*"(MIT License|Apache 2\.0|BSD|GPL|ISC License)"
# Should be: MIT, Apache-2.0, BSD-2-Clause, GPL-3.0-only, ISC

# SPDX expression syntax
(?i)(MIT|Apache-2\.0|GPL-\d|BSD)\s+(OR|AND)\s+(MIT|Apache-2\.0|GPL-\d|BSD)

# Proprietary markers
(?i)(all rights reserved|proprietary|confidential|not for distribution)
```

---

## 2. License Type Taxonomy

Understanding the obligations of each license type is essential for determining
compatibility and risk. This taxonomy covers the major categories encountered in
software dependency trees.

### Permissive Licenses

Permissive licenses allow virtually unrestricted use, modification, and
redistribution. They impose minimal obligations, typically limited to preserving
copyright notices and disclaimers.

| License | SPDX ID | Key Obligations | Risk Level |
|---|---|---|---|
| MIT | `MIT` | Preserve copyright notice and license text in copies | **Minimal** |
| BSD 2-Clause (Simplified) | `BSD-2-Clause` | Preserve copyright notice and disclaimer | **Minimal** |
| BSD 3-Clause (New) | `BSD-3-Clause` | Preserve copyright + no endorsement clause | **Minimal** |
| ISC | `ISC` | Preserve copyright notice (functionally equivalent to MIT) | **Minimal** |
| Apache 2.0 | `Apache-2.0` | Preserve NOTICE file, explicit patent grant, state changes if modified | **Minimal** |
| Zlib | `Zlib` | Preserve copyright in source; binary distribution unrestricted | **Minimal** |
| Unlicense | `Unlicense` | None (public domain dedication) | **Minimal** |
| CC0 1.0 | `CC0-1.0` | None (public domain dedication) | **Minimal** |
| 0BSD | `0BSD` | None (no attribution required) | **Minimal** |
| WTFPL | `WTFPL` | None | **Minimal** (may not be recognized by legal departments) |

**Apache-2.0 special considerations**:
- The explicit patent grant protects downstream users from patent litigation by contributors
- A `NOTICE` file must be preserved and distributed with the software
- Modified files must carry prominent notice of changes
- The patent grant terminates if a user initiates patent litigation against the licensor

### Weak Copyleft Licenses

Weak copyleft licenses require that modifications to the licensed component
itself be shared under the same terms, but do not extend that obligation to
the larger work that uses the component.

| License | SPDX ID | Copyleft Scope | Key Obligations | Risk Level |
|---|---|---|---|---|
| LGPL 2.1 | `LGPL-2.1-only` / `LGPL-2.1-or-later` | Library-level: modifications to the LGPL code must be shared | Dynamic linking is safe; static linking triggers copyleft for the combined work | **Moderate** |
| LGPL 3.0 | `LGPL-3.0-only` / `LGPL-3.0-or-later` | Library-level: same as LGPL 2.1 but inherits GPLv3 patent and anti-tivoization terms | Must allow users to replace the LGPL library with a modified version | **Moderate** |
| MPL 2.0 | `MPL-2.0` | File-level: modified files must be shared, but new files in the same project are unaffected | Modified MPL-licensed files must be released under MPL; your own files are unrestricted | **Low-Moderate** |
| Eclipse Public License 2.0 | `EPL-2.0` | Module-level: modifications to EPL code must be shared | Includes patent grant; secondary license option allows GPLv2 compatibility | **Moderate** |
| Common Development and Distribution License | `CDDL-1.0` | File-level: similar to MPL | Modified files must be released under CDDL; incompatible with GPL | **Moderate** |

**LGPL static vs. dynamic linking**:
- **Dynamic linking** (shared libraries, `.so`, `.dll`, `.dylib`): The LGPL component is a separate work. The larger application's license is unaffected. This is the safe path.
- **Static linking** (compiled into the binary): The result is a combined work, and the LGPL copyleft extends to the combined work. The user must be able to relink with a modified version of the LGPL library.
- **Language-specific nuances**: In Go, all dependencies are statically linked. In Rust, dependencies are statically linked by default. In Java, the classpath exception (in Oracle's LGPL interpretation) and module system boundaries are relevant. In JavaScript/Python, imports are effectively dynamic.

### Strong Copyleft Licenses

Strong copyleft licenses require that any work that incorporates, links to, or
is derived from the licensed code must itself be distributed under the same
copyleft terms. This is the most restrictive category.

| License | SPDX ID | Copyleft Trigger | Key Obligations | Risk Level |
|---|---|---|---|---|
| GPL 2.0 | `GPL-2.0-only` / `GPL-2.0-or-later` | Distribution of binaries or combined works | Source code of the entire combined work must be available under GPL | **High** |
| GPL 3.0 | `GPL-3.0-only` / `GPL-3.0-or-later` | Distribution of binaries or combined works | Same as GPLv2, plus explicit patent grant, anti-tivoization provisions, and installation information requirement | **High** |
| AGPL 3.0 | `AGPL-3.0-only` / `AGPL-3.0-or-later` | Network interaction (users accessing the software over a network triggers copyleft) | Must offer source code to users who interact with the software over a network, even without distribution | **Very High** |

**AGPL-3.0 is the highest-risk copyleft license for SaaS and cloud applications**:
- Unlike GPL, AGPL considers providing access to software over a network as "conveying" the work
- A proprietary SaaS product that uses an AGPL library server-side must release its entire source code under AGPL
- This applies even if the AGPL component is a small utility library
- CRITICAL finding for any proprietary or commercial SaaS product

**GPL "or later" implications**:
- `GPL-2.0-or-later` means the user can choose GPLv2 or any later version (including GPLv3)
- `GPL-2.0-only` restricts to GPLv2 exclusively
- This distinction affects compatibility: GPLv2-only is incompatible with Apache-2.0, but GPLv3 is compatible

### Non-Open-Source and Restricted Licenses

These licenses are not recognized by the Open Source Initiative (OSI) and impose
restrictions that conflict with open-source principles or create significant
legal risk.

| License | SPDX ID | Restriction | Risk Level |
|---|---|---|---|
| Server Side Public License | `SSPL-1.0` | Offering the software as a service requires releasing the entire service stack under SSPL | **Very High** — effectively prohibits SaaS use without full source release |
| Commons Clause | N/A (modifier) | Prohibits selling the software "as a service" or "substantially" as the product | **High** — ambiguous definition of "substantially" creates legal uncertainty |
| Business Source License | `BUSL-1.1` | Software is source-available but not open-source; converts to open-source after a change date | **High** — review the change date, usage grant, and permitted uses carefully |
| Elastic License 2.0 | `Elastic-2.0` | Cannot provide the software as a managed service; cannot modify and remove license restrictions | **High** — targeted at preventing SaaS competition |
| Proprietary / Commercial | N/A | Terms defined by vendor agreement; may restrict distribution, modification, or number of users | **High** — requires legal review of specific terms |
| No License / Unknown | N/A | All rights reserved by default under copyright law; no permission to use, modify, or distribute | **CRITICAL** — using unlicensed code is copyright infringement |
| Creative Commons (non-software) | `CC-BY-4.0`, etc. | CC licenses are designed for creative works, not software; CC themselves recommend against using CC for software | **Moderate** — not legally problematic per se but indicates confusion |

### Severity Classification by License Type

| License Encountered | Project Type: Proprietary | Project Type: Permissive OSS | Project Type: Copyleft OSS |
|---|---|---|---|
| MIT, BSD, ISC, Apache-2.0 | INFO | INFO | INFO |
| MPL-2.0 | LOW (file-level copyleft) | LOW | INFO |
| LGPL (dynamic linking) | LOW | LOW | INFO |
| LGPL (static linking) | HIGH | MEDIUM | INFO |
| GPL (any version) | CRITICAL | HIGH (must relicense) | INFO (if compatible version) |
| AGPL | CRITICAL | CRITICAL (unless AGPL project) | MEDIUM (unless AGPL project) |
| SSPL | CRITICAL | CRITICAL | CRITICAL |
| Commons Clause / BUSL | HIGH (review terms) | HIGH | HIGH |
| No License / Unknown | CRITICAL | CRITICAL | CRITICAL |

---

## 3. Compatibility Matrix

License compatibility determines whether code under one license can be combined
with code under another license in the same project. Incompatibility means the
licenses impose contradictory obligations, making legal distribution impossible.

### Key Compatibility Rules

| Combination | Compatible? | Notes |
|---|---|---|
| MIT + BSD + ISC + Apache-2.0 | Yes | All permissive licenses are mutually compatible |
| Apache-2.0 + GPL-2.0-only | **No** | Apache-2.0 patent clause cannot be accommodated by GPLv2 |
| Apache-2.0 + GPL-3.0 | Yes | GPLv3 explicitly addresses the Apache patent clause |
| MIT + GPL (any) | Yes | MIT code can be incorporated into GPL projects (one-way) |
| GPL-2.0-only + GPL-3.0-only | **No** | "Only" versions are mutually incompatible |
| GPL-2.0-or-later + GPL-3.0 | Yes | "Or later" allows upgrading to GPLv3 |
| LGPL + proprietary (dynamic link) | Yes | Dynamic linking preserves separation |
| LGPL + proprietary (static link) | Conditional | Must allow relinking with modified LGPL library |
| MPL-2.0 + GPL | Yes | MPL 2.0 includes explicit GPL compatibility (Section 3.3) |
| MPL-2.0 + Apache-2.0 | Yes | Both are permissive at the project level |
| CDDL + GPL | **No** | CDDL and GPL have incompatible copyleft terms |
| AGPL + non-AGPL (in same service) | **No** (for proprietary) | AGPL infects the entire service |
| EPL-2.0 + GPL-2.0 | Yes (with secondary license) | EPL-2.0 allows designation of GPL as secondary license |

### One-Way Compatibility

Many license combinations are one-way compatible. Code under a permissive
license can flow into a copyleft project, but not the reverse.

```
Direction of code flow:

  MIT/BSD/ISC ──────> GPL ──────> AGPL
       │                │
       │                └──> Cannot flow back to MIT/BSD/ISC
       │
       └──> Apache-2.0 ──> GPL-3.0 (but NOT GPL-2.0-only)
```

- Code licensed under MIT can be used in a GPL project. The combined work is GPL.
- Code licensed under GPL cannot be extracted and used in an MIT project.
- Apache-2.0 code can flow into GPL-3.0 but not GPL-2.0-only.

### What to Check

| # | Checkpoint | Severity |
|---|---|---|
| 3.1 | All dependency licenses are identified (no "UNKNOWN" or missing license fields) | **HIGH** |
| 3.2 | No GPL/AGPL dependencies in a proprietary project | **CRITICAL** |
| 3.3 | No AGPL dependencies in any project that is not itself AGPL-licensed | **CRITICAL** |
| 3.4 | No Apache-2.0 dependencies combined with GPL-2.0-only code | **HIGH** |
| 3.5 | No CDDL dependencies combined with GPL code | **HIGH** |
| 3.6 | SSPL and Commons Clause dependencies are flagged for legal review | **HIGH** |
| 3.7 | Multi-licensed dependencies have the chosen license explicitly documented | **MEDIUM** |
| 3.8 | License compatibility has been evaluated for the full transitive dependency tree, not just direct dependencies | **MEDIUM** |
| 3.9 | No dependencies with "custom" or non-standard licenses that have not been reviewed by legal | **HIGH** |
| 3.10 | Dependencies with different versions of the same license family (e.g., GPL-2.0-only vs. GPL-3.0-only) are checked for cross-compatibility | **MEDIUM** |

---

## 4. Obligation Tracking

Each license imposes specific obligations on users of the licensed code. Failure
to meet these obligations constitutes a license violation, which can range from
a polite request to comply to full-scale litigation.

### Attribution Obligations

| License | Attribution Required? | What to Include | Where |
|---|---|---|---|
| MIT | Yes | Copyright notice + license text | In source distributions and in a NOTICE/ATTRIBUTION file for binary distributions |
| BSD-2-Clause | Yes | Copyright notice + disclaimer | Same as MIT |
| BSD-3-Clause | Yes | Copyright notice + disclaimer + non-endorsement clause | Same as MIT; additionally, must not use contributor names for endorsement |
| ISC | Yes | Copyright notice + license text | Same as MIT |
| Apache-2.0 | Yes | NOTICE file contents + license text + state changes to modified files | NOTICE file must be included in all distributions; modifications must be marked |
| Zlib | Yes (source only) | Copyright notice in source form | Binary distributions have no attribution requirement |

### Source Disclosure Obligations

| License | When Triggered | What Must Be Disclosed | Severity if Missing |
|---|---|---|---|
| GPL-2.0/3.0 | Distribution of binaries or combined works | Complete corresponding source code of the combined work | **CRITICAL** |
| AGPL-3.0 | Network interaction (providing access to the software over a network) | Complete corresponding source code, accessible to network users | **CRITICAL** |
| LGPL-2.1/3.0 | Static linking | Object files or source sufficient to allow relinking with a modified LGPL library | **HIGH** |
| MPL-2.0 | Distribution of modified MPL-licensed files | Source code of modified MPL-licensed files only (not the entire project) | **MEDIUM** |
| EPL-2.0 | Distribution of modifications | Source code of modifications to EPL-licensed modules | **MEDIUM** |

### Patent Grant Tracking

| License | Patent Grant? | Scope | Termination Trigger |
|---|---|---|---|
| Apache-2.0 | Yes, explicit | Covers patents that would be infringed by the contribution | Terminates if the licensee initiates patent litigation against the licensor related to the software |
| GPL-3.0 | Yes, implicit | Each contributor grants a non-exclusive patent license | Terminates on license violation |
| MPL-2.0 | Yes, explicit | Covers patents necessarily infringed by contributions | Terminates if licensee initiates patent litigation |
| MIT, BSD, ISC | No explicit grant | Patent rights are ambiguous (may be implied but not guaranteed) | N/A |
| EPL-2.0 | Yes, explicit | Covers patents necessarily infringed by the contribution | Terminates on patent litigation |

### Notice Preservation

| # | Checkpoint | Severity |
|---|---|---|
| 4.1 | Copyright notices in source files are preserved, not removed or altered | **HIGH** |
| 4.2 | License header comments in source files are preserved when files are copied or vendored | **HIGH** |
| 4.3 | Apache-2.0 NOTICE files are included in all distributions (source and binary) | **MEDIUM** |
| 4.4 | A THIRD-PARTY-NOTICES or ATTRIBUTION file lists all dependencies with their licenses | **MEDIUM** |
| 4.5 | Binary distributions include license texts for all bundled dependencies | **MEDIUM** |
| 4.6 | Modified files carry a prominent notice stating that they were changed (required by Apache-2.0, GPL) | **LOW** |

### What to Look For

```regex
# Copyright notices in source
(?i)copyright\s+(\(c\)\s*)?\d{4}

# License headers
(?i)(licensed under|released under|distributed under)\s+(the\s+)?(MIT|Apache|BSD|GPL|LGPL|MPL|ISC)

# SPDX license headers (best practice)
SPDX-License-Identifier:\s*\S+

# Apache NOTICE file references
(?i)NOTICE(\.(txt|md))?$

# Third-party attribution files
(?i)(THIRD.?PARTY|ATTRIBUTION|NOTICES|CREDITS)(\.(txt|md))?$
```

---

## 5. Vendored Code Tracking

Vendored code is source code copied directly into the repository rather than
installed via a package manager. It requires special attention because it exists
outside the normal dependency management workflow and its license obligations
persist regardless of how it was obtained.

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 5.1 | Vendored code directories (`vendor/`, `third_party/`, `external/`, `lib/vendored/`) are identified and inventoried | **MEDIUM** |
| 5.2 | Each vendored component has its license file preserved alongside the code | **HIGH** |
| 5.3 | Each vendored component has its origin, version, and license documented in a manifest (e.g., `vendor/MANIFEST.md`, `third_party/README.md`) | **MEDIUM** |
| 5.4 | Vendored code has not been modified without documenting the changes | **MEDIUM** |
| 5.5 | Vendored code licenses are compatible with the project license (same rules as Section 3) | **CRITICAL** (if incompatible) |
| 5.6 | Vendored code is not stale: versions are tracked and updated when security patches are released | **HIGH** |
| 5.7 | Code snippets copied from external sources (Stack Overflow, blog posts, GitHub Gists) are attributed with source URL and license | **MEDIUM** |
| 5.8 | Stack Overflow code (licensed CC-BY-SA 4.0) compatibility with the project license has been evaluated | **MEDIUM** |
| 5.9 | Auto-generated code (protobuf stubs, OpenAPI clients, code generators) includes the generator's license attribution if required | **LOW** |
| 5.10 | Vendor directories are included in license scanning tool configuration (not excluded by default) | **MEDIUM** |

### Vendored Code Manifest Format

A well-maintained vendor manifest should include:

```markdown
# Third-Party Code Manifest

| Component | Version | License | Source | Modified? | Notes |
|---|---|---|---|---|---|
| lodash | 4.17.21 | MIT | https://github.com/lodash/lodash | No | Vendored for offline builds |
| xxhash | 0.8.2 | BSD-2-Clause | https://github.com/Cyan4973/xxHash | Yes (see CHANGES.md) | Patched for ARM alignment |
| jwt-decode | 3.1.2 | MIT | https://github.com/auth0/jwt-decode | No | Subset: only decode function |
```

### What to Look For

```regex
# Common vendored directories
(?i)^(vendor|third_party|external|vendored|deps|lib/vendor)/

# Code origin comments (positive indicators)
(?i)(copied from|ported from|based on|adapted from|source:|origin:|originally from)\s+https?://

# Stack Overflow attribution
(?i)(stackoverflow\.com|stackexchange\.com)/questions/\d+

# Code generator markers
(?i)(auto.?generated|do not edit|generated by|@generated)
```

---

## 6. SPDX Identifier Usage

The Software Package Data Exchange (SPDX) standard provides unambiguous, machine-readable
license identifiers. Using SPDX consistently enables automated license scanning, policy
enforcement, and compliance reporting.

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 6.1 | Package manifest uses SPDX license identifiers (not free-form text) | **LOW** |
| 6.2 | Source files include SPDX-License-Identifier headers where the project convention requires them | **LOW** |
| 6.3 | Multi-license expressions use SPDX syntax: `OR` for choice, `AND` for combined obligations | **LOW** |
| 6.4 | License exceptions use SPDX syntax: `GPL-2.0-only WITH Classpath-exception-2.0` | **LOW** |
| 6.5 | Deprecated SPDX identifiers are replaced with current equivalents (`GPL-2.0` -> `GPL-2.0-only` or `GPL-2.0-or-later`) | **LOW** |
| 6.6 | Custom or non-SPDX licenses use `LicenseRef-` prefix: `LicenseRef-Proprietary-Acme` | **LOW** |

### Common SPDX Mistakes

| Incorrect | Correct SPDX | Why |
|---|---|---|
| `MIT License` | `MIT` | SPDX identifiers do not include the word "License" |
| `Apache 2.0` | `Apache-2.0` | Must use exact SPDX ID with hyphen |
| `BSD` | `BSD-2-Clause` or `BSD-3-Clause` | "BSD" alone is ambiguous (which BSD variant?) |
| `GPL` | `GPL-2.0-only` or `GPL-3.0-only` | Must specify version and only/or-later |
| `MIT/Apache-2.0` | `MIT OR Apache-2.0` | Use `OR` operator, not slash |
| `MIT AND Apache-2.0` | `MIT AND Apache-2.0` | Correct if both apply simultaneously; use `OR` if the user can choose |
| `GPLv2+` | `GPL-2.0-or-later` | Use standard suffix |

### SPDX License Header Format

```
// SPDX-License-Identifier: MIT
```

```
// SPDX-License-Identifier: Apache-2.0
```

```
// SPDX-License-Identifier: GPL-3.0-only
```

```
// SPDX-License-Identifier: MIT OR Apache-2.0
```

---

## 7. Multi-Licensed Dependency Handling

Many popular libraries offer multiple license options (e.g., "MIT OR Apache-2.0").
When a dependency is multi-licensed, the consumer can choose which license to
accept. This choice should be deliberate, documented, and consistent.

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 7.1 | Multi-licensed dependencies are identified in the dependency tree | **MEDIUM** |
| 7.2 | The chosen license for each multi-licensed dependency is documented (e.g., in a `.licensefinder.yml` or ATTRIBUTION file) | **MEDIUM** |
| 7.3 | The chosen license is compatible with the project license and other dependencies | **HIGH** |
| 7.4 | When a dependency offers GPL-2.0 OR MIT, the permissive option (MIT) is chosen for proprietary projects | **HIGH** |
| 7.5 | License scanning tools are configured to resolve multi-licensed dependencies to the chosen license, not flag them as ambiguous | **LOW** |

### Common Multi-Licensed Packages

| Package/Ecosystem | Common Dual License | Recommended Choice (Proprietary) |
|---|---|---|
| Rust standard ecosystem | `MIT OR Apache-2.0` | Either (both permissive); Apache-2.0 for patent grant |
| many npm packages | `MIT OR ISC` | Either (functionally identical) |
| Qt | `LGPL-3.0 OR GPL-2.0 OR Commercial` | Commercial (if available) or LGPL with dynamic linking |
| MySQL Connector | `GPL-2.0 OR Commercial` | Commercial license required for proprietary use |
| Oracle Berkeley DB | `AGPL-3.0 OR Commercial` | Commercial license required for proprietary use |

### What to Look For

```regex
# Multi-license expressions in manifests
(?i)"license"\s*:\s*"\(?\s*(MIT|Apache|BSD|GPL|ISC|MPL)\s+(OR|AND)\s+(MIT|Apache|BSD|GPL|ISC|MPL)"

# Parenthesized SPDX expressions
\(MIT OR Apache-2\.0\)

# License scanner resolution configs
(?i)(permitted|approved|allowed|accepted).*licenses?
```

---

## 8. License Scanning Tools

Automated license scanning is essential for any project with more than a handful
of dependencies. Manual license tracking does not scale and misses transitive
dependencies.

### Tool Comparison

| Tool | Ecosystems | CI Integration | SBOM Output | Cost |
|---|---|---|---|---|
| **FOSSA** | All major | GitHub, GitLab, Jenkins, CircleCI, Bitbucket | SPDX, CycloneDX | Free tier + paid |
| **license-checker** (npm) | Node.js only | Any (CLI) | JSON, CSV | Free (open source) |
| **licensefinder** (Pivotal) | Ruby, Node, Python, Go, Java, .NET, Rust | Any (CLI) | CSV, text | Free (open source) |
| **Snyk** | All major | GitHub, GitLab, Jenkins, Bitbucket, Azure | SPDX | Free tier + paid |
| **Trivy** (Aqua) | Container images, filesystems, Git repos | GitHub Actions, GitLab CI, any CLI | SPDX, CycloneDX | Free (open source) |
| **OSS Review Toolkit (ORT)** | All major | Any (CLI) | SPDX, CycloneDX, custom | Free (open source) |
| **ScanCode** | Language-agnostic (file-level scanning) | Any (CLI) | SPDX, CycloneDX | Free (open source) |
| **pip-licenses** | Python only | Any (CLI) | JSON, CSV, text | Free (open source) |
| **go-licenses** (Google) | Go only | Any (CLI) | CSV, template | Free (open source) |
| **cargo-license** | Rust only | Any (CLI) | JSON, text | Free (open source) |

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 8.1 | A license scanning tool is configured for the project | **HIGH** |
| 8.2 | The tool scans both direct and transitive dependencies | **MEDIUM** |
| 8.3 | The tool covers all ecosystems used in the project (not just the primary language) | **MEDIUM** |
| 8.4 | An approved license allowlist or denied license blocklist is configured | **MEDIUM** |
| 8.5 | Unknown or undetectable licenses trigger a review, not a silent pass | **HIGH** |
| 8.6 | The tool runs on every PR or commit (not just periodic manual scans) | **MEDIUM** |
| 8.7 | Vendored code is included in the scan scope | **MEDIUM** |
| 8.8 | Tool output is stored as an artifact for audit trail purposes | **LOW** |
| 8.9 | False positives are documented with justification, not silently suppressed | **MEDIUM** |
| 8.10 | The tool generates or contributes to an SBOM | **LOW** |

### What to Look For

```regex
# FOSSA configuration
(?i)\.fossa\.(yml|yaml|json)

# LicenseFinder configuration
(?i)(\.license.?finder|license_finder)\.(yml|yaml)
(?i)doc/dependency_decisions\.yml

# Snyk configuration
(?i)(\.snyk|snyk\.(json|yml))

# Trivy license scanning
(?i)trivy.*--scanners\s+license

# License allowlist/blocklist patterns
(?i)(allowed.?licenses?|permitted.?licenses?|blocked.?licenses?|denied.?licenses?|license.?allowlist|license.?blocklist)

# CI integration indicators
(?i)(license.?check|license.?scan|license.?audit|compliance.?check)
```

---

## 9. Automated CI Enforcement

License compliance must be enforced in CI/CD pipelines to prevent non-compliant
dependencies from entering the codebase. Relying solely on periodic audits
leaves gaps where non-compliant code ships to production.

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 9.1 | License scanning runs as a required CI check on pull requests | **HIGH** |
| 9.2 | CI fails (non-zero exit code) when a non-approved license is detected | **HIGH** |
| 9.3 | New dependencies trigger a license review step (automated or manual) | **MEDIUM** |
| 9.4 | License scan results are posted as PR comments or check annotations | **LOW** |
| 9.5 | A documented process exists for requesting an exception to the license policy | **MEDIUM** |
| 9.6 | Exception approvals are recorded in version control (not just Slack messages) | **MEDIUM** |
| 9.7 | The license policy is defined as code (not just a wiki page) so it is versioned and auditable | **MEDIUM** |
| 9.8 | CI pipeline includes SBOM generation as a build artifact | **LOW** |
| 9.9 | License compliance status is visible in a dashboard or periodic report | **LOW** |
| 9.10 | Breaking changes to the license policy trigger re-evaluation of existing dependencies | **MEDIUM** |

### CI Integration Examples

**FOSSA in GitHub Actions**:

```yaml
- name: FOSSA Analyze
  uses: fossas/fossa-action@v1
  with:
    api-key: ${{ secrets.FOSSA_API_KEY }}

- name: FOSSA Test (policy check)
  uses: fossas/fossa-action@v1
  with:
    api-key: ${{ secrets.FOSSA_API_KEY }}
    run-tests: true  # Fails if policy violations exist
```

**license-checker in npm projects**:

```json
// package.json
{
  "scripts": {
    "license-check": "license-checker --onlyAllow 'MIT;ISC;BSD-2-Clause;BSD-3-Clause;Apache-2.0;0BSD;CC0-1.0;Unlicense' --excludePrivatePackages"
  }
}
```

```yaml
# GitHub Actions
- name: License check
  run: npx license-checker --onlyAllow 'MIT;ISC;BSD-2-Clause;BSD-3-Clause;Apache-2.0;0BSD;CC0-1.0;Unlicense' --excludePrivatePackages
```

**licensefinder in CI**:

```yaml
# GitHub Actions
- name: License check
  run: |
    gem install license_finder
    license_finder --decisions-file doc/dependency_decisions.yml
```

```yaml
# doc/dependency_decisions.yml (licensefinder policy)
---
- - :permit
  - MIT
  - :who: legal-team
    :why: Approved permissive license
    :when: 2024-01-15
- - :permit
  - Apache-2.0
  - :who: legal-team
    :why: Approved permissive license with patent grant
    :when: 2024-01-15
- - :restrict
  - AGPL-3.0
  - :who: legal-team
    :why: Incompatible with proprietary distribution
    :when: 2024-01-15
```

**Trivy license scanning**:

```yaml
- name: Trivy license scan
  uses: aquasecurity/trivy-action@0.28.0
  with:
    scan-type: fs
    scanners: license
    severity: UNKNOWN,HIGH,CRITICAL
    exit-code: 1
```

**go-licenses in Go projects**:

```yaml
- name: Check Go licenses
  run: |
    go install github.com/google/go-licenses@latest
    go-licenses check ./... --disallowed_types=forbidden,restricted
```

---

## Review Procedure Summary

When evaluating license compatibility:

1. **Identify the project license**: Confirm a LICENSE file exists at the repo root
   and matches the package manifest. Determine whether the project is proprietary,
   permissive open-source, or copyleft open-source.

2. **Enumerate all dependencies**: Examine package manifests and lock files. Include
   transitive dependencies. Flag any dependencies with missing or unknown licenses.

3. **Classify each dependency license**: Use the taxonomy in Section 2 to categorize
   each license. Pay special attention to copyleft (GPL, AGPL, LGPL) and non-open
   (SSPL, Commons Clause, BUSL) licenses.

4. **Check compatibility**: Use the matrix in Section 3 to verify that every
   dependency license is compatible with the project license. Flag incompatible
   combinations.

5. **Verify obligation compliance**: For each license in the dependency tree,
   confirm that the project meets attribution, source disclosure, notice
   preservation, and patent grant obligations (Section 4).

6. **Audit vendored code**: Check that all vendored code has its license
   preserved, its origin documented, and its license compatible with the project
   (Section 5).

7. **Evaluate tooling**: Confirm that license scanning is automated, integrated
   into CI, and configured with an appropriate policy (Sections 8-9).

8. **Check SPDX and documentation**: Verify SPDX identifiers are used correctly,
   multi-licensed dependencies have documented choices, and a THIRD-PARTY-NOTICES
   file exists (Sections 6-7).

9. **Classify each finding** using the severity tables in each section.

10. **Prioritize remediation**: CRITICAL findings (GPL/AGPL in proprietary code,
    unlicensed dependencies) first. HIGH findings (missing license file, no scanning
    tool) second. MEDIUM and below are compliance improvements.
