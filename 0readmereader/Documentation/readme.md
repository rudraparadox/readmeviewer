# SBOM-Gen — Complete Project Documentation

**SIH 2024 Problem Statement 1449 · NTRO (National Technical Research Organisation)**

> This document explains the entire tool — what it does, how it works internally, and how to
> present it. Each section has two parts: **Plain English** (say this out loud to judges) and
> **Technical details** (to back it up / answer deep questions).

---

## Table of Contents

1. [What is this tool? (30-second pitch)](#1-what-is-this-tool)
2. [The problem we are solving](#2-the-problem-we-are-solving)
3. [How it works — the whole story in plain words](#3-how-it-works--the-whole-story-in-plain-words)
4. [Tech stack at a glance](#4-tech-stack-at-a-glance)
5. [System architecture](#5-system-architecture)
6. [Deep dive — module by module](#6-deep-dive--module-by-module)
   - 6.1 Input handling (zip / folder / GitHub / Docker)
   - 6.2 Ecosystem detection & lock-file parsing
   - 6.3 Binary artifact analysis
   - 6.4 Source / import analysis (reachability)
   - 6.5 Vulnerability engine
   - 6.6 Context analyzer (red flags + remediation)
   - 6.7 SBOM generation (CycloneDX / SPDX)
   - 6.8 Reports (HTML & PDF)
   - 6.9 Policy engine (CI/CD gating)
   - 6.10 The `sbomgen` CLI
   - 6.11 Auth & multi-tenancy
   - 6.12 Frontend dashboard
7. [Key design decisions](#7-key-design-decisions)
8. [How to run it](#8-how-to-run-it)
9. [Demo script for judges (7-minute live walkthrough)](#9-demo-script-for-judges)
10. [Likely judge questions & suggested answers](#10-likely-judge-questions--suggested-answers)
11. [Glossary](#11-glossary)

---

## 1. What is this tool?

### Plain English

> Imagine software is like a recipe. When a company builds software, it usually uses **hundreds
> of ready-made ingredients** — these are called "libraries" or "dependencies" (for example, a
> JavaScript project might use a library called `lodash`, a Python project might use `requests`).
>
> This tool is like a **smart nutrition label + expiry checker** for software. You hand it your
> project (a zip file, a folder, or a GitHub link) and it automatically:
>
> 1. **Reads the recipe** — lists every ingredient (library) with its **exact version**.
> 2. **Checks the expiry date** — checks every ingredient against a database of known
>    **vulnerabilities** (security holes) and marks the dangerous ones in **red**.
> 3. **Tells you exactly how to fix it** — gives you the precise command to upgrade the bad
>    ingredient.
> 4. **Prints the official "nutrition label"** — exports it in the government-standard SBOM
>    formats (CycloneDX and SPDX) that security teams and regulations require.
>
> And the best part — **it works fully offline**, which is critical for sensitive government
> (NTRO) environments that cannot access the internet.

### Technical (one-liner)

A web-based, polyglot Software Bill of Materials (SBOM) generator + vulnerability scanner that
parses lock files and binary artifacts, cross-references components against a bundled offline
CVE dataset plus the live OSV API, flags anomalies, computes remediation, and emits
CycloneDX 1.5 / SPDX 2.3 / SPDX 3.0 SBOMs, HTML and PDF reports — with a web dashboard and a
CI/CD CLI.

---

## 2. The problem we are solving

### Plain English

- Modern software is rarely written from scratch — **80–90% of a modern application is
  third-party code** (open-source libraries).
- These libraries have security holes (CVEs) discovered over time. If you don't know **exactly
  which version** of which library you are using, you cannot know if you are vulnerable.
- This became a legal/regulatory requirement: governments and big organizations now **require**
  every piece of software to ship with an SBOM — a machine-readable "ingredients list".
- NTRO's problem statement adds a special twist: the tool must work **offline**, because
  classified/intelligence environments cannot talk to public vulnerability databases over the
  internet.

### The criteria the judges will score on

| Criterion | What our tool does |
|---|---|
| **Automation** | Drag a `.zip` or paste a GitHub URL → full SBOM + vulnerability report in one click. |
| **Granularity** | Lock-file based parsing → exact versions, transitive dependencies, integrity hashes, direct/transitive/dev flags. 8+ ecosystems. |
| **Accuracy** | Parses *resolved lock files* (ground truth of what was installed), not guesswork from source code. |
| **Red-flag anomalies** | Bundled 836-record CVE dataset + live OSV lookup; typosquatting, shadow deps, deprecated packages, unpinned versions, license conflicts. |
| **Context** | CVE details, CVSS score & vector, CISA Known-Exploited / malicious markers, **reachability** (is it actually used in your code?), and the exact fix command. |
| **Ease of use** | Web dashboard with interactive graph, filters, one-click CycloneDX/SPDX/PDF export, and a CLI for pipelines. |
| **Offline / in-house** | Bundled offline dataset → full scan with zero internet. |

---

## 3. How it works — the whole story in plain words

> Follow one scan from start to finish. This is your "elevator pitch" of the pipeline.

**Step 0 — You give it something to scan.**
You can drop a **zip file**, type a **server folder path**, paste a **GitHub URL**, or even a
**Docker image name**. The tool accepts all four.

**Step 1 — It finds the "receipts" (lock files).**
Modern package managers leave behind *lock files* — files that record exactly what was
installed and which version. Examples: `package-lock.json` (Node.js), `requirements.txt` /
`poetry.lock` (Python), `pom.xml` (Java), `go.mod` (Go), `Cargo.lock` (Rust), `composer.lock`
(PHP), `Gemfile.lock` (Ruby), `packages.lock.json` (.NET), `build.gradle` (Android/Java).
The tool walks the whole project and detects **every** ecosystem present at once
(polyglot support). If no lock file exists, it falls back to reading the manifests or analysing
the source code.

**Step 2 — It builds the "ingredient list" (components).**
For each ecosystem it parses the lock file and creates a *component* for every library:
its **name**, **exact version**, **package URL (purl)**, **integrity hashes** (so the file can be
verified byte-for-byte), and whether it is a **direct** dependency (you asked for it) or a
**transitive** dependency (it came bundled inside something else). It also records the
**dependency edges** — "A depends on B, B depends on C" — so you can draw the full tree.

**Step 3 — It also inspects "packaged goods" (binary artifacts).**
If the project contains already-built files — Python wheels, Java JARs, Windows `.exe`/`.dll`,
Linux `.so` — it extracts their metadata too, so shipped binaries land in the SBOM even when you
don't have the source.

**Step 4 — It looks inside your actual code.**
It reads your source code's **import statements** (`import requests`, `require('lodash')`,
`#include <openssl/ssl.h>`, `import com.fasterxml.jackson...`). This powers two smart features:
- **Reachability** — "is this vulnerable library *actually used* in my code, or just sitting in
  the dependency tree?"
- **Shadow dependencies** — libraries imported in code but not declared in any manifest
  (a red flag — could be a hidden/hijacked package).

**Step 5 — It checks every ingredient against known vulnerabilities.**
Every `name@version` is looked up in:
- a **bundled offline dataset** of 836 curated advisories across 8 ecosystems (works with no
  internet), and
- the **live OSV (Open Source Vulnerabilities) API** (a Google-backed public database), with a
  local SQLite cache so repeat scans are instant and offline-friendly.

**Step 6 — It adds context and red flags.**
For every vulnerability found it attaches: CVE/ID, **CVSS score** (0–10) and severity
(CRITICAL/HIGH/MEDIUM/LOW), **CISA Known-Exploited** and **malicious-package** markers, whether
it is **reachable from your code**, and the **exact upgrade command** to fix it.
It also flags *anomalies*: typosquatted package names, deprecated/yanked versions, unpinned
versions, unmaintained packages, license conflicts (e.g., using GPL inside a commercial product),
and version drift (the same package at multiple versions).

**Step 7 — It generates the official SBOM + reports.**
It emits standards-compliant **CycloneDX 1.5**, **SPDX 2.3** and **SPDX 3.0** JSON — with the
vulnerabilities, hashes, and dependency relationships **embedded inside**, so the SBOM is
consumable end-to-end. It also generates a self-contained **HTML report** and a **PDF report**
with an overall "Supply-Chain Health Score".

**Step 8 — You see it in a beautiful dashboard.**
The web UI shows summary cards, an **interactive dependency graph** (vulnerable nodes glow red,
attack paths highlighted), a filterable component table, an anomaly feed, a code browser with
per-line vulnerability markers, and one-click export/download buttons.

**Step 9 — (Optional) You automate it.**
The same engine runs from a command line (`sbomgen`), so CI/CD pipelines can scan on every
commit and **fail the build** if a critical vulnerability appears.

---

## 4. Tech stack at a glance

| Layer | Technology | Why |
|---|---|---|
| Backend API | **Python · FastAPI** | Fast async REST API, Pydantic validation, easy background threading |
| Frontend | **React + Vite** | Single-page dashboard, fast builds |
| Dependency graph | **d3-hierarchy** (custom SVG) | Interactive tree/attack-path visualization |
| CycloneDX output | **cyclonedx-python-lib** | Official library — no hand-rolled schema, standards-compliant |
| SPDX output | Hand-built (JSON) | Full control over relationships/vulnerabilities embedding |
| Persistence | **SQLite** | Job store, OSV cache, Maven cache, auth, audit log — no DB server needed |
| Vulnerability data | **OSV API** (live) + **bundled seed_data.json** (offline) | Both worlds: freshness + air-gapped operation |
| PDF reports | **reportlab** | A4, print-ready executive reports |
| Packaging | **PyInstaller** | Single `.exe` — no Python/Node install needed |
| Auth | **JWT + bcrypt** | Multi-tenant organisations, roles, API tokens, audit log |

---

## 5. System architecture

```
┌───────────────────────────────────────────────────────────────┐
│               React Dashboard  (frontend/, Vite)              │
│  Uploader · Summary cards · Dependency graph · Table          │
│  Anomalies · Code browser · Licenses · Binaries               │
│  Context panel (CVSS + fix command) · Export buttons          │
└────────────────────────────┬──────────────────────────────────┘
                             │ REST (same-origin /api)
┌────────────────────────────▼──────────────────────────────────┐
│                  FastAPI  (backend/app/main.py)               │
│   POST /api/scan  (zip | path | github_url | docker) → job_id │
│   GET  /api/scan/{id}                 (poll status + result)  │
│   GET  /api/scan/{id}/cyclonedx|spdx|spdx30|report|report.pdf │
│   GET  /api/scan/{id}/tree|file|validate|diff  ...            │
│   Background worker threads (daemon), SQLite job store        │
└────────────────────────────┬──────────────────────────────────┘
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
   ┌───────────┐      ┌────────────┐      ┌──────────────┐
   │ scanner/  │      │  vuln/     │      │  context/    │
   │ detect    │      │  osv.py    │      │  analyzer.py │
   │ 18 parsers│      │  cache.py  │      │  severity    │
   │ runner.py │      │  seed_data │      │  remediation │
   │ ast_      │      │  (836 CVEs)│      │  reachability│
   │ scanner   │      │  SQLite    │      │  anomalies   │
   │ binary_   │      │  cache     │      │  licenses    │
   │ parser    │      └────────────┘      └──────────────┘
   └───────────┘                 ▼
                         ┌──────────────┐
                         │  sbom/       │  CycloneDX 1.5
                         │  generator   │  SPDX 2.3 / 3.0
                         │  hashing.py  │  file hashes
                         └──────────────┘
```

### Data flow (one sentence)

Upload → detect ecosystems → parse lock files → build `Component` list (name, exact version,
purl, hashes, direct/transitive, children/parents) → AST import analysis → vulnerability
enrichment (offline seed + live OSV) → anomaly & license checks → generate CycloneDX / SPDX /
SPDX-3.0 → store result in SQLite keyed by job id → UI polls until complete.

---

## 6. Deep dive — module by module

> File paths are relative to `backend/app/`. For each module: first the **Plain English**, then
> the **Technical details** you can quote when judges probe deeper.

### 6.1 Input handling — how a scan is started

**Files:** `api/routes.py`, `scanner/runner.py`, `scanner/docker_parser.py`

#### Plain English

You have four ways to feed the tool, and it figures out which one you meant:

1. **Zip file** — drag & drop any project zip. It safely extracts it into a temporary folder
   (guarding against "zip-slip" path-traversal attacks) and scans it.
2. **Server folder path** — type a folder on the machine where the tool runs.
3. **GitHub URL** — paste `https://github.com/owner/repo`; the tool does a shallow `git clone`
   and scans the downloaded copy.
4. **Docker image** — pull/extract a container image and scan the OS packages + language
   libraries inside it.
5. **Single binary** — if the uploaded file is not a zip but a compiled artifact (`.whl`,
   `.jar`, `.exe`, `.dll`, `.so`), it scans the binary directly.

#### Technical details

- `POST /api/scan` accepts `multipart/form-data` with `file`, `path`, `github_url`, or
  `docker_images`, plus `offline_only` and `fetch_licenses` flags. Precedence:
  docker → github → path → file.
- The scan runs in a **daemon thread** so a big repository (2,000+ dependencies) doesn't block
  the API. The client polls `GET /api/scan/{id}` until `status` becomes `complete` or `error`.
- Job statuses: `pending → scanning → complete | error`.
- Results are persisted to SQLite (`jobs.db`, `%LOCALAPPDATA%\sih1449\`) with `INSERT OR REPLACE`,
  so **scan history survives a restart**.
- Zip extraction (`_extract_zip`) resolves every member path and refuses anything that escapes
  the destination directory (zip-slip protection).
- GitHub clones use `git clone --depth 1 --quiet` with a 120-second timeout.
- Uploaded projects are stored (`projects.py`) so the **code browser** (with per-line
  vulnerability flags) keeps working after a restart.

---

### 6.2 Ecosystem detection & lock-file parsing

**Files:** `scanner/node.py` (detection), `scanner/runner.py` (dispatch), plus one parser per
ecosystem.

#### Plain English

The tool first **walks the whole folder** looking for "signatures" — known lock-file names.
Whatever it finds tells it which package managers the project uses. A project can use several at
once (e.g., a Python backend + a JavaScript frontend) and the tool scans **all of them in
parallel**.

It then picks the **most authoritative file** per ecosystem. Lock files are preferred over plain
manifests because lock files contain **exact, resolved versions** — the ground truth of what
would actually be installed.

#### The detection table (how the tool knows which ecosystem is which)

| Lock file / manifest | Ecosystem |
|---|---|
| `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `package.json` | npm (Node.js) |
| `requirements.txt`, `poetry.lock`, `Pipfile.lock`, `pyproject.toml`, `setup.py` | pypi (Python) |
| `pom.xml` | maven (Java) |
| `go.mod` (+ `go.sum`) | go (Golang) |
| `Cargo.lock` | cargo (Rust) |
| `composer.lock` | composer (PHP) |
| `Gemfile.lock` | bundler (Ruby) |
| `packages.lock.json`, `project.assets.json` | nuget (.NET) |
| `build.gradle`, `build.gradle.kts` | gradle |
| `Podfile.lock` | cocoapods (iOS) |
| `conan.lock`, `conanfile.txt` | conan (C++) |
| `conda-lock.yml`, `environment.yml` | conda |
| `vcpkg.json`, `vcpkg-lock.json` | vcpkg (C++) |
| `packages.list`, `installed`, `status` (dpkg) | linux (OS packages) |

**Technical details — what each parser captures:**

- **npm** (`node.py`): handles `package-lock.json` **v1, v2 and v3** (v1 uses a nested
  `node_modules` tree, v2/v3 use a flat `packages` map); `yarn.lock` **v1 and v2/berry**;
  `pnpm-lock.yaml` v5/v6/v9. Captures SRI `integrity` hashes (e.g. `sha512-...`), direct/dev
  flags from the root package's `dependencies`/`devDependencies`, and dependency edges by
  resolving the nearest `node_modules` install.
- **Python** (`python_parser.py`): `requirements.txt` (recursively follows `-r` includes,
  resolves ranges against the PyPI JSON API), `poetry.lock` (hashes from `files[]`, edges from
  `dependencies`), `Pipfile.lock` (default vs develop groups), `pyproject.toml` (PEP 621 + legacy
  poetry style), and `setup.py` (parsed with Python's AST — **never executes** your setup script).
- **Maven** (`maven_resolver.py` — the most powerful parser): assembles the *effective POM*
  (inheritance, property `${}` resolution, `dependencyManagement`, imported BOMs), resolves the
  **full transitive tree** with conflict resolution (nearest-wins), prunes `test/provided`
  scopes, and even fetches POMs from Maven Central online (SQLite-cached) or from a bundled
  offline tree dataset. Attaches licenses, homepage, and SHA-1 hashes. Falls back to a simple
  `java_parser.py` if needed.
- **Go** (`go_parser.py`): parses `require` blocks, marks `// indirect` lines as transitive, and
  attaches **go.sum `h1:` SHA-256 integrity hashes**.
- **Cargo (Rust)**: parses `Cargo.lock`, attaches the crate `checksum` (SHA-256), builds edges
  by crate name.
- **Composer (PHP)**: direct/dev flags from `composer.json`, hashes from `dist.sha256`/`shasum`,
  edges from `require`/`require-dev`.
- **Bundler (Ruby)**: reads the `GEM`/`GIT`/`DEPENDENCIES` sections of `Gemfile.lock`; nested
  indentation defines dependency edges.
- **NuGet (.NET)**: `packages.lock.json` (direct/transitive via `type` field, `contentHash`
  SHA-512) and `project.assets.json` (edges from `dependencies`, `sha512` hashes); directness
  cross-checked against `<PackageReference>` in `.csproj` files.
- **Gradle**: manifest-only parsing of `implementation/api/...` configurations plus **version
  catalogs** (`libs.versions.toml`) with `version.ref` resolution. (No transitive resolution —
  listed as a known limitation.)
- **Cocoapods / Conan / Conda / vcpkg / Linux**: parse their lock files or package stores; e.g.
  dpkg `status` stanzas → `pkg:deb` components.

---

### 6.3 Binary artifact analysis

**Files:** `scanner/binary_parser.py`

#### Plain English

Some projects ship **already-built binaries** without full source. This tool can read those
binaries directly and pull out their "ingredients":

- **Python wheel** (`.whl`) — reads the embedded `.dist-info/METADATA` for name/version and its
  required dependencies.
- **Java JAR** (`.jar/.war/.ear`) — reads `META-INF/maven/*/pom.properties` and the embedded
  `pom.xml` for coordinates and dependencies.
- **Windows PE** (`.exe/.dll`) — reads the **import table** to list the Windows DLLs the binary
  links against (filtering out OS noise like `kernel32.dll`).
- **Linux ELF** (`.so/.elf`) — reads the `DT_NEEDED` entries to list linked shared libraries.

#### Technical details

- Identification is by file extension first, then **magic bytes** (`MZ` → PE; `\x7fELF` → ELF;
  `PK` zip sniffed for `.dist-info` vs `META-INF` → wheel vs jar).
- PE import parsing uses the `pefile` library, with a raw byte-scan regex fallback.
- ELF parsing is **hand-implemented** (parses the ELF header, locates `.dynamic`/`.dynstr`
  sections, reads `DT_NEEDED`), supporting 32/64-bit and endianness, with a regex fallback.
- Each binary gets a **SHA-256 file hash**; every extracted dependency is linked back as a child
  (`parents=[binary]`).
- In the dashboard these appear under a dedicated **Binaries** tab with the components extracted
  from them.

---

### 6.4 Source / import analysis (reachability, shadow & unused deps)

**Files:** `scanner/ast_scanner.py`, `scanner/runner.py`

#### Plain English

The tool **reads your actual code** to find what's really being used. It scans import statements
across 10+ languages. Three powerful outputs come from this:

1. **Reachability** — if a vulnerable library is in your dependency tree but your code never
   imports it, the risk is lower. The tool marks each vulnerability as **reachable from code** or
   not, and shows exactly which `file:line` uses it.
2. **Shadow dependencies** — code imports a package that is **not declared in any manifest**.
   This is a red flag (it could be a malicious/hijacked package) — flagged as a `shadow_dependency`
   and still CVE-scanned.
3. **Unused dependencies** — a package is declared in the manifest but **never imported**.
   Flagged as `unused_dependency` (dead weight / supply-chain bloat).

#### Technical details

- Per-language static analysis: Python uses real `ast.parse` (never executes code); JS uses
  regex for `require`/`import`/dynamic `import()`; Go, Java, C#, Ruby, Rust, PHP use regex; and
  C/C++ maps `#include <header>` → library name (`zlib.h`→zlib, `openssl/ssl.h`→openssl,
  `curl/curl.h`→curl, `sqlite3.h`→sqlite3, `boost/...`→boost, etc.).
- Stdlib / built-in / system-header lists keep false positives low (Python stdlib, Node core
  modules, JDK prefixes, .NET BCL prefixes, C system headers).
- Import names are mapped to real package names (`cv2`→`opencv-python`, `PIL`→`Pillow`,
  `sklearn`→`scikit-learn`, `bs4`→`beautifulsoup4`).
- Import locations are attached to components and shown in the **code browser** with per-line
  vulnerability markers.

---

### 6.5 Vulnerability engine

**Files:** `vuln/osv.py`, `vuln/cache.py`, `vuln/seed_data.json`

#### Plain English

This is the "expiry checker". Every component `name@version` is checked against:

1. **Bundled offline dataset** — 836 curated security advisories across 8 ecosystems
   (npm, PyPI, Maven, RubyGems, Go, Packagist, crates.io, NuGet), so the whole tool works
   **completely offline** — a hard requirement for NTRO.
2. **Live OSV API** — Google's public vulnerability database, queried at scan time, with
   **everything cached in local SQLite**, so repeat scans are instant and don't hammer the
   network.

Version matching is precise: advisories specify an *introduced* version (first affected,
inclusive) and a *fixed* version (safe, exclusive). If your version falls in that window, you're
vulnerable.

#### Technical details

- OSV ecosystem mapping: npm→npm, pypi→PyPI, maven→Maven, go→Go, cargo→crates.io,
  composer→Packagist, bundler→RubyGems, nuget→NuGet, gradle→Maven.
- Live lookups run through `query_osv_batch` with an 8-thread pool; cached entries short-circuit
  with zero network. The single-package `OSV_API` endpoint is used (the `/v1/querybatch` constant
  exists but is currently unused).
- CVSS parsing handles both legacy (vector string ending in `:9.8`) and modern (numeric score +
  vector) OSV shapes; the **highest** score wins when multiple severities exist.
- Severity labels: CRITICAL ≥ 9.0, HIGH ≥ 7.0, MEDIUM ≥ 4.0, LOW > 0, else UNKNOWN. Falls back
  to `database_specific.severity` (GitHub/npm style labels) when no CVSS is present.
- **KEV flag** (CISA Known-Exploited) is detected heuristically from advisory metadata
  (searches for CISA+KEV / "exploited" markers) — best-effort, not a live CISA feed.
- **Malicious packages** are flagged when the advisory ID starts with `MAL-` (OSV's malware
  prefix).
- The bundled `seed_data.json`: 836 records, 819 unique IDs, 153 distinct packages; 228 PyPI,
  201 npm, 120 Maven, 91 RubyGems, 69 Go, 55 Packagist, 47 crates.io, 25 NuGet. Real CVE numbers
  live in the `aliases` field (e.g. `GHSA-29mw-wpgm-hmr9` aliases `CVE-2020-28500` for lodash).
- Semver comparison ignores pre-release/build suffixes for range matching
  (`4.17.21-rc.1` compares equal to `4.17.21`).

---

### 6.6 Context analyzer — red flags, remediation & stats

**Files:** `context/analyzer.py`

#### Plain English

Knowing "you have a vulnerability" is only half the story. This module adds the **context** that
makes the report actionable:

- **Severity + CVSS**: a score from 0–10 and a vector string, so you know *how* dangerous.
- **Fix command**: the exact, copy-pasteable command to upgrade, in the right package manager:
  - Python → `pip install --upgrade requests>=2.32.0`
  - Node → `npm install lodash@4.17.21`
  - Go → `go get github.com/foo/bar@v1.2.3`
  - Rust → `cargo update -p serde --precise 1.0.200`
  - PHP → `composer require vendor/pkg:1.2.3`
  - Ruby → `bundle update pkg -v 1.2.3`
  - .NET → `dotnet add package Foo --version 1.2.3`
  - Java/Gradle → "bump ... to ... in your build file".
- **Red-flag anomalies** — the tool actively hunts for suspicious patterns:
  - **Typosquatting** — a package name almost identical to a popular one (e.g. `requests` vs
    `reqests`).
  - **Unpinned version** — versions like `*`, `^1.0`, `>=2.0` (you don't know what you'll get).
  - **Dependency confusion** — packages from `file:`, `workspace:`, `git+` or private registry
    prefixes, or package names that don't exist in the public registry (hijack risk).
  - **Unmaintained** — no release in 2+ years (HIGH if 5+ years).
  - **Suspicious provenance** — a package published very recently (≤90 days) that isn't popular.
  - **Deprecated / yanked** — the version is marked yanked (PyPI) or deprecated (npm).
  - **Version drift** — the same direct package present at multiple versions.
  - **Copyleft license conflicts** — e.g. GPL/AGPL/SSPL inside a permissive/commercial project.
  - **Shadow / unused dependencies** — from the AST analysis (see 6.4).

#### Technical details

- Direct-dependency checks gate most anomaly checks to keep false positives low.
- License/supplier metadata is enriched from bundled `seed_licenses.json` (offline-safe), live
  PyPI/npm registry data, and (optionally) the ClearlyDefined API.
- Copyleft severity: AGPL/SSPL → HIGH, GPL → MEDIUM, others → LOW. If the *project* license is
  permissive, a copyleft dependency raises a HIGH "license conflict".
- Final `stats` dict powers the dashboard cards: total/direct/transitive components, vulnerable
  components, unique vulnerabilities, per-severity counts, `max_severity`, KEV count, malicious
  count, reachable-vulnerability count, binary count, lock-file count, import/binary component
  counts, and anomaly count.

---

### 6.7 SBOM generation — CycloneDX & SPDX

**Files:** `sbom/generator.py`, `sbom/spdx30.py`, `sbom/hashing.py`

#### Plain English

The whole point of the tool is to produce the **official machine-readable ingredient list**.
It emits three industry-standard formats:

- **CycloneDX 1.5** (JSON) — the modern favorite for security tooling.
- **SPDX 2.3** (JSON) — the other major standard, widely used by legal/licensing teams.
- **SPDX 3.0** (JSON) — the next-generation format.

Crucially, these aren't just "a list of names" — each SBOM has the **vulnerabilities, integrity
hashes, and dependency relationships embedded inside**, so whoever receives the file gets the
full security picture, not just the ingredient names.

#### Technical details

- CycloneDX is built with the official **`cyclonedx-python-lib`** (never a hand-rolled schema):
  metadata component, `purl` per component, hashes, licenses, scope (OPTIONAL for dev), custom
  `Property`s for direct/dev/source-file/locations, a root `Dependency` linking every direct
  component, and **embedded `Vulnerability` objects** (rating with CVSS method chosen from the
  vector prefix — CVSS:4.0 / 3.1 / 3.0, recommendation "Fixed in X", and `kev`/`malicious`/
  `reachable` properties). A `files[]` array (SHA-256 content hashes) is post-injected.
- SPDX 2.3 is hand-built: one `SPDXRef-Package` per component with checksums, `externalRefs`
  (purl PACKAGE-MANAGER ref + license ref), `DEPENDS_ON` relationships per edge, vulnerabilities
  encoded as SECURITY/advisory external refs linked by `AFFECTS`, an NTIA-minimum root package
  (`DESCRIBES`, `DEPENDS_ON`, `CONTAINS` each hashed file), and `externalDocumentRefs` +
  `PREREQUISITE_FOR` for linked SBOMs. Deterministic `documentNamespace`.
- SPDX 3.0 (core profile): JSON-LD with `Package`/`Vulnerability`/`Relationship` elements,
  `usesDependency`, `affects`, `rootElements`, `verifiedUsing` hashes.
- `hashing.py` computes **SHA-256 + SHA-1** for every project file in one pass (thread-pooled),
  capped (2 MiB/file, 20,000 files by default) to stay fast.
- SBOM generation is reproducible: options mirror ms-sbom-tool's `-gt/-nsb/-nsu`
  (reproducible timestamp, namespace base/unique) and serial numbers become deterministic
  `uuid5` when a namespace base is set.

---

### 6.8 Reports — HTML & PDF

**Files:** `report/generator.py`, `report/pdf.py`

#### Plain English

Not everyone wants raw JSON. The tool produces a **printable HTML report** and a **PDF report**
that an executive can read: a big **Supply-Chain Health Score** circle, a severity bar, stat
cards, a vulnerability findings table, an anomaly table, the full component inventory, binary
artifacts, and a license summary.

#### Technical details

- **Health score formula:** `100 − (CRITICAL × 30) − (HIGH × 12) − (MEDIUM × 5) − (LOW × 2)`,
  clamped 0–100; labelled (e.g. Excellent / Good / At Risk).
- HTML report is a single self-contained file (dark themed, `@media print` stylesheet that flips
  to white/black for printing), with CVE links to NVD, per-finding flag badges
  (`CISA KNOWN EXPLOITED` / `MALICIOUS` / `REACHABLE`), "Used In file:line" links, "Fixed In"
  and the remediation command.
- PDF uses **reportlab**, A4, latin-1-safe text cleaning, per-row severity coloring, and page
  numbers. Served at `GET /api/scan/{id}/report.pdf` with filename `sbom-report-{id}.pdf`.

---

### 6.9 Policy engine — gate your CI/CD

**Files:** `policy/engine.py`, `policy/store.py`

#### Plain English

Organizations want to **enforce rules**, not just view reports. The policy engine lets an admin
define rules, and the scan result gets a `passed` / `failed` verdict with a list of violations.
Examples: "fail if any CRITICAL vulnerability", "fail if any CISA Known-Exploited issue", "fail
if a malicious package is present", "block this specific package/version", "block copyleft
licenses".

#### Technical details

- Rules: `max_severity`, `max_vulnerabilities`, `fail_on_kev`, `fail_on_malicious`,
  `blocked_package` (name/version aware, with allowlist), `blocked_license` /
  `allowed_license`.
- Default policy ships with the tool: max severity `critical`, max vulnerabilities `0`,
  `fail_on_kev: true`, `fail_on_malicious: true`, blocked licenses `AGPL-3.0`, `AGPL-3.0-only`,
  `AGPL-3.0-or-later`, `SSPL-1.0`.
- Policies are stored per-organisation in SQLite; the web layer evaluates the org's effective
  policy, the CLI uses the default.

---

### 6.10 The `sbomgen` CLI

**Files:** `backend/sbomgen.py`

#### Plain English

The whole engine is available from the command line, which is how it plugs into **CI/CD
pipelines** (GitHub Actions, GitLab CI, Jenkins...). Run it on every commit; if a critical
vulnerability appears, the build **fails** with exit code 2.

```powershell
# Scan a folder, offline, emit CycloneDX
python -m sbomgen scan D:\repos\my-app --offline --format cyclonedx -o sbom.json

# Scan a zip, full JSON result
python -m sbomgen scan app.zip --format json

# Scan a GitHub repo and produce a PDF
python -m sbomgen scan https://github.com/owner/repo --format pdf -o report.pdf

# Gate your pipeline
python -m sbomgen scan D:\repos\my-app --fail-on high    # exits 2 if anything >= HIGH
```

#### Technical details

- Subcommands: `scan`, `validate` (verify files against manifest), `format` (structural
  validation), `conformance` (NTIA minimum), `aggregate` (merge 2+ SBOMs), `redact` (strip file
  detail), `sign` (Ed25519), `verify`, `verify-catalog`.
- Formats: `json` (full ScanResult), `cyclonedx`, `spdx`, `spdx30`, `html`, `pdf`.
- Exit codes: `0` success, `1` scan/IO error, `2` `--fail-on` gate tripped or validation failed.
- A Python library API is also exported (`scan()`, `validate()`, `aggregate()`, `redact()`,
  `sign()`, `verify_signature()`) for embedding.
- `--config` supports JSON config files with `$(ENV_VAR)` expansion; `--telemetry-file` writes
  structured scan telemetry.

---

### 6.11 Auth & multi-tenancy

**Files:** `auth/security.py`, `auth/store.py`, `auth/deps.py`, `api/auth_routes.py`

#### Plain English

The web app supports **multiple organisations** with real accounts, so each team only sees its
own scans. Users have roles (admin / analyst / viewer), admins can manage users and generate
API tokens for automation, and every important action is written to an **audit log**.

#### Technical details

- bcrypt (10 rounds) password hashing; HS256 JWTs with 12-hour expiry; signing key from
  `SBOM_SECRET_KEY` env or a persisted secret file.
- Service-account API tokens (`sbom_<48 hex>`) are stored as SHA-256 hashes and accepted in
  `Authorization: Bearer` alongside JWTs.
- SQLite tables: `organizations`, `users`, `audit_log`. Org-scoped scan visibility is enforced
  on every scan endpoint.

---

### 6.12 Frontend dashboard

**Files:** `frontend/src/App.jsx`, `api.js`, `graph.js`, `components/*`

#### Plain English

The dashboard is where everything comes together visually. After a scan completes you get:

- **Summary cards** — total/direct/transitive components, lock files, binaries, import-detected
  deps, vulnerable components, unique vulnerabilities, reachable vulns, KEV count, anomalies —
  plus an animated health-score ring and severity distribution bar.
- **Dependency graph** — an interactive tree. Red nodes = vulnerable, attack paths from the root
  to a vulnerable node are highlighted in red, import-detected and binary nodes get dashed
  rings. You can zoom, pan, expand/collapse, and control depth.
- **Components table** — searchable + filterable by severity, ecosystem, dependency type
  (direct/transitive/dev), vulnerable-only, and origin (lockfile/manifest/import/binary).
- **Context panel** — click any component: where it's used in code, who depends on it and what it
  depends on, integrity hashes, and each vulnerability with CVSS, flags, and the **copy-paste
  fix command**.
- **Code browser** — browse the uploaded source; vulnerable lines are marked with severity
  colors.
- **Anomalies / Licenses / Binaries tabs** — red flags feed, license compliance view with
  copyleft warnings, and extracted binary artifacts.
- **Scan history** — all previous scans, with **compare (diff)** between two scans
  (added/removed components, new/resolved vulnerabilities, before/after score) and **merge**
  into a combined SBOM.

#### Technical details

- Polling: `pollScan` checks `GET /api/scan/{id}` every **1.5 s** until terminal state; a 900 ms
  step-timer drives the 4-step progress animation.
- The graph is **d3-hierarchy** (not a force-directed library): a synthetic root node, children
  edges from each component's `children`, cycle protection via ancestor checks, and **attack
  paths** computed by BFS from root + backtracking from every vulnerable node.
- Severity colors: CRITICAL/HIGH/MEDIUM/LOW mapped to a fixed palette; vulnerable nodes pulse.
- The frontend build (`frontend/dist`) is served directly by FastAPI (SPA catch-all with
  `html=True`), and `vite.config.js` proxies `/api` → `:8000` in dev.
- A PyInstaller windowed build launches a **pywebview (EdgeChromium)** window with a JS bridge
  that saves exports straight to `~/Downloads`, falling back to the default browser.

---

## 7. Key design decisions

| Decision | Why it makes the submission strong |
|---|---|
| **Lock files over source scanning** | Lock files record *exact resolved versions* — the ground truth. No guessing. |
| **Polyglot detection in one pass** | One scan covers mixed projects (Python + JS + Java...) automatically. |
| **Binary artifacts counted too** | Wheels/JARs/PE/ELF are parsed directly — shipped binaries land in the SBOM even without source. |
| **Lock-file-less fallback** | AST/import analysis still surfaces the third-party packages *actually used*. |
| **Bundled offline dataset + live OSV + SQLite cache** | Offline-safe for NTRO, fresh when online, fast on repeat scans. |
| **Official CycloneDX library** | Standards compliance without hand-rolled schema errors. |
| **Vulnerabilities embedded in the SBOM** | The SBOM is consumable end-to-end — one file carries the whole picture. |
| **Source-aware analysis** | Reachability, shadow deps and unused deps turn a list into *actionable risk*. |
| **Background jobs + polling** | 2,000+ dependency repos don't block the API. |
| **Policy engine + CLI** | CI/CD gating — the tool isn't just a UI, it's a pipeline guard. |
| **Everything in SQLite** | No database server to install; portable, restart-safe. |

### Known limitations (be honest if asked)

- Gradle is manifest-only (no transitive resolution).
- Yarn lock parsing produces a flat list (no dependency edges) — directness comes from
  `package.json`.
- KEV detection is a heuristic from advisory metadata, not a live CISA feed.
- The OSV "batch" lookup currently performs one request per component (8 concurrent threads)
  rather than a single `/querybatch` call — cached entries avoid re-querying.
- Semver range matching ignores pre-release/build suffixes.

---

## 8. How to run it

### 1. Backend

```powershell
cd backend
pip install -r requirements.txt
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

### 2. Frontend (development, hot reload)

```powershell
cd frontend
npm install
npm run dev          # http://localhost:5173 (proxies /api → :8000)
```

For a single-binary deployment, build once and FastAPI serves it:

```powershell
cd frontend && npm run build
# then open http://127.0.0.1:8000/
```

### 3. Tests

```powershell
cd backend
python tests\test_e2e.py          # end-to-end API tests (npm + PyPI + multi-app)
python tests\test_regression.py   # severity/robustness regressions
python tests\test_binary.py       # wheel/JAR/PE/ELF + lock-file-less detection
python tests\test_cli.py          # sbomgen CLI (formats, exit codes, --fail-on)
cd .. && cd frontend && npm run build
```

### 4. CLI

```powershell
cd backend
python -m sbomgen scan D:\repos\my-app --offline --format cyclonedx -o sbom.json
python -m sbomgen scan app.zip --format json
python -m sbomgen scan https://github.com/owner/repo --format pdf -o report.pdf
```

### 5. Standalone `.exe` (no installs needed)

```powershell
cd frontend && npm run build
cd ..
pip install pyinstaller
python -m PyInstaller SBOM-Gen.spec --noconfirm   # -> dist\SBOM-Gen.exe
```

> The vuln cache DB defaults to `%LOCALAPPDATA%\sih1449\vulns.db` (set `SBOM_DB_PATH` to
> override). Some drives/antivirus setups break SQLite locking, hence C: by default.

---

## 9. Demo script for judges

> Goal: ~7 minutes. Setup beforehand: backend running at `http://127.0.0.1:8000`, frontend
> built. The demo uses `D:\1SIH_Hackthon\samples\multi-app` (a polyglot demo containing both
> `package-lock.json` and `requirements.txt`).

### 0:00 — The pitch (30 s)
> "Software today is 80–90% third-party libraries. Nobody can manually track thousands of
> versions. Our tool, SBOM-Gen, automatically generates the complete ingredient list — a
> Software Bill of Materials — checks every ingredient against known vulnerabilities, tells you
> exactly how to fix them, and works fully offline. NTRO's exact requirement."

### 0:30 — Launch a scan (30 s)
1. Open `http://127.0.0.1:8000`.
2. In the uploader, choose the **folder path** input.
3. Paste `D:\1SIH_Hackthon\samples\multi-app`.
4. Click **Scan**. Point out the progress steps.
> "Notice it detected two ecosystems at once — Node and Python. Watch the 4-step pipeline:
> detect → parse → analyze → report."

### 1:30 — Summary cards + graph (90 s)
1. When complete, gesture at the **summary cards**.
> "Here's the headline: X components across two ecosystems, Y vulnerable, Z reachable from
> code, and an overall health score."
2. Switch to the **graph** tab. Hover/pan/zoom.
> "Each node is a library. Red nodes are vulnerable. Notice how the vulnerable node sits deep in
> the dependency tree — it came in as a transitive dependency. We traced the attack path from
> the root to it."

### 3:00 — Click a vulnerable package (90 s)
1. Click `lodash@4.17.12` (or whichever vulnerable node shows).
2. Point to the **context panel**:
   - CVSS 9.8, CRITICAL, CVE ID (CVE-2019-10744).
   - **"Reachable in code"** badge → "we proved it's actually imported in `src/index.js`".
   - The **fix command** → copy it.
> "Vulnerabilities alone are noise. We tell you *where it is*, *whether you're actually using
> it*, and the *exact one-line fix*. This is what 'contextual guidance' in the problem statement
> means."

### 4:30 — Export SBOMs (60 s)
1. Click **CycloneDX** and **SPDX**.
> "These are the government-standard formats. Open the file and show the embedded
> vulnerabilities, integrity hashes, and dependency relationships — a single consumable file,
> not just a name list."
2. Click **Report (PDF)** and open the PDF.
> "And here's the executive view — a health score and a findings table that a non-technical
> stakeholder can read."

### 5:30 — Prove offline mode (60 s)
1. Toggle **offline mode** on and re-scan.
> "This is the NTRO differentiator — no internet. All vulnerability data is bundled inside the
> tool (836 advisories across 8 ecosystems). Flip the switch and it still finds the same
> critical issue."

### 6:30 — CI/CD (30 s, optional if time permits)
> "The same engine runs headless. `python -m sbomgen scan ... --fail-on high` — exit code 2
> fails the pipeline. One click in the UI, one command in the pipeline — same result."

### Closing line
> "To summarize: automatic, polyglot, offline-capable, standards-compliant SBOM generation with
> actionable, reachability-aware vulnerability context — built for exactly NTRO's problem."

---

## 10. Likely judge questions & suggested answers

**Q1. What is an SBOM and why does it matter?**
An SBOM (Software Bill of Materials) is a machine-readable list of every component in a piece of
software, with versions, hashes and relationships. It matters because ~90% of modern software is
third-party code — if you don't know what's in your software, you can't know if you're
vulnerable. NTRO and similar agencies need SBOMs for software they procure, to audit supply
chains and respond to incidents quickly.

**Q2. Why lock files instead of scanning the source?**
A lock file records the *exact resolved* versions that were actually installed, including every
transitive dependency. Source scanning only tells you what the code *asks* for — the range
`^4.0.0` could resolve to many versions. Accuracy of versions and the complete transitive tree
is only possible from lock files.

**Q3. What if a project has no lock file?**
We fall back, in order: (1) parse the manifest (`package.json`, `requirements.txt`,
`pyproject.toml`...) and resolve version ranges via the registry API; (2) analyse the actual
imports in the code with our AST scanner. Accuracy drops slightly, but we still surface the
third-party packages actually used. We also flag unpinned versions as an anomaly.

**Q4. How does offline mode work?**
All vulnerability data ships inside the tool — a bundled dataset of 836 curated advisories
across 8 ecosystems (npm, PyPI, Maven, RubyGems, Go, Packagist, crates.io, NuGet), plus an
offline license metadata set. Version-range matching, severity, CVSS and fix commands all work
from this local data. Online mode layers the live OSV API on top, cached in SQLite.

**Q5. How accurate is the version matching?**
Advisories declare an introduced (inclusive) and fixed (exclusive) version. We compare your
version against that window numerically. We parsed real lock-file formats (npm v1–v3, yarn v1/v2,
poetry, go.sum hashes...) and verified against real npm projects, so the versions themselves are
exact.

**Q6. What makes this different from free tools like `npm audit` / `pip-audit` / Dependabot?**
Those are single-ecosystem. Ours is a *unified, polyglot* tool covering 18+ formats across 8+
ecosystems plus binary artifacts, producing *standard* SBOMs (CycloneDX/SPDX) that regulators
require — not just a vulnerability list. We add reachability, shadow/unused dependency
detection, typosquatting, license conflicts, an offline mode for air-gapped environments, an
executive health-score report, and a policy gate for CI/CD.

**Q7. How do you determine "reachable"?**
Our AST scanner parses the actual import statements in your source (Python, JS, Go, Java, C#,
Ruby, PHP, Rust, C/C++). If a vulnerable package's name matches an import, we mark every
vulnerability for that package as reachable and record the exact file:line. Otherwise we still
report it, but flag it as not reachable from code — lower priority.

**Q8. How is this handled in CI/CD?**
Two ways: the `sbomgen` CLI with `--fail-on <level>` (exits 2 when the worst vulnerability
crosses the threshold), and a policy engine (`max_severity`, `max_vulnerabilities`,
`fail_on_kev`, `fail_on_malicious`, blocked packages/licenses) evaluated on every scan. Your
pipeline can block the merge, not just warn.

**Q9. What about licenses?**
Every component gets a declared license where available (from lock file, bundled dataset, live
registry, or ClearlyDefined). We classify permissive vs copyleft, flag copyleft (AGPL/SSPL/GPL)
as anomalies, raise a HIGH "license conflict" when a permissive project pulls in copyleft code,
and render a license compliance dashboard with counts.

**Q10. Can it handle compiled/binary-only software?**
Yes. We parse Python wheels (embedded dist-info), Java JARs (META-INF/pom.properties), Windows
PE files (import table) and Linux ELF files (DT_NEEDED) — extracting the components and linked
libraries with file SHA-256 hashes, so a shipped binary still produces an SBOM without source.

**Q11. How does the health score work?**
`100 − 30×(critical count) − 12×(high) − 5×(medium) − 2×(low)`, clamped to 0–100. It's a
single glanceable number an executive can grasp immediately, backed by the detailed findings
table.

**Q12. What are "shadow dependencies" and why do they matter?**
A shadow dependency is a package your code imports but that isn't declared in any manifest. It's
a red flag because it bypasses your normal dependency management — it could be a
typosquat/malicious/hijacked package, and no standard tool will update or audit it. We flag it
as a MEDIUM anomaly and CVE-scan it anyway.

**Q13. Did you build the SPDX/CycloneDX outputs from scratch?**
CycloneDX uses the official `cyclonedx-python-lib` to guarantee schema compliance. SPDX 2.3/3.0
are hand-built JSON — SPDX gives us full control to embed vulnerabilities, file hashes and
`DEPENDS_ON`/`AFFECTS` relationships exactly as required.

**Q14. How do you handle multi-module / huge monorepos?**
Detection walks the whole tree and scans every ecosystem. Maven resolves every `pom.xml`
(multi-module) and merges/dedupes by purl. Scans run in background daemon threads with bounded
parallelism (configurable, default 8) and file hashing caps (2 MiB/file, 20k files) — so even
2,000+ dependency repos don't block the API.

**Q15. What about false positives / the reliability of the KEV flag?**
The KEV flag is a best-effort heuristic from advisory metadata (CISA/KEV/exploited markers),
clearly a limitation. Core vulnerability matching relies on OSV's structured range data rather
than keywords, which keeps matching precise. Our offline dataset is curated (836 records, real
GHSA/PYSEC/GO/RUSTSEC IDs with CVE aliases).

---

## 11. Glossary

| Term | Easy meaning |
|---|---|
| **SBOM** | Software Bill of Materials — a machine-readable list of all components in a piece of software, like an ingredient label. |
| **Dependency** | A library your project uses. |
| **Direct dependency** | A library you declared/asked for yourself. |
| **Transitive dependency** | A library that arrived *inside* another library (a dependency of a dependency). |
| **Lock file** | A file a package manager writes that records the *exact* versions installed (e.g. `package-lock.json`, `Cargo.lock`). |
| **Manifest** | The file where you *declare* dependencies, usually with version ranges (e.g. `package.json`, `requirements.txt`). |
| **Component** | One library/package in the SBOM (name + version + metadata). |
| **purl** | Package URL — a universal way to name a package, e.g. `pkg:npm/lodash@4.17.12`. |
| **Integrity hash** | A cryptographic fingerprint (SHA-1/256/512) that lets you verify a file hasn't been tampered with. |
| **CVE** | Common Vulnerabilities and Exposures — a public ID for a known security flaw (e.g. `CVE-2019-10744`). |
| **CVSS** | Common Vulnerability Scoring System — a 0–10 score of how dangerous a flaw is, plus a vector explaining why. |
| **OSV** | Open Source Vulnerabilities — Google's public vulnerability database, queried via an API. |
| **KEV** | CISA Known Exploited Vulnerabilities — flaws that are *actually being exploited in the wild*. |
| **Reachability** | Whether a vulnerable package is actually imported/used in your code (vs just present in the tree). |
| **Shadow dependency** | A package imported in code but not declared in any manifest — a red flag. |
| **Typosquatting** | Registering a package with a name almost identical to a popular one (e.g. `reqests`) to trick users. |
| **CycloneDX** | One of the two major SBOM standards (JSON), loved by security tooling. |
| **SPDX** | The other major SBOM standard, widely used for licensing. |
| **SBOM conformance / NTIA minimum** | The minimum required fields every SBOM should have (NTIA "Minimum Elements"). |
| **Attack path** | The chain of dependencies from your root project down to a vulnerable package. |
| **Supply-chain health score** | A single 0–100 number summarizing how many and how severe the issues are. |

---

*Documentation generated for SBOM-Gen (SIH1449). For the most up-to-date run instructions and
repository layout, see `README.md`.*
