# SBOM-Gen — Project Overview & Understanding Guide

**SIH 2024 · Problem Statement 1449 · NTRO (National Technical Research Organisation)**

> A plain-English guide to what this project is and how it works — written so one person can
> read it, understand the whole tool, and confidently answer a judge's questions.
> For the full technical deep-dive, see `DOCUMENTATION.md`.

---

## 1. What is this tool?

**One line:** SBOM-Gen automatically generates a **Software Bill of Materials (SBOM)** for any
software project, checks every library against known **vulnerabilities**, red-flags risky
patterns, tells you **exactly how to fix problems**, and exports everything in government
standard formats — all while working **fully offline**.

### What is an SBOM?

An SBOM (Software Bill of Materials) is the **"ingredients list" of software**. Just like a food
package must list what's inside, modern software must list every third-party library it uses,
with exact versions and integrity hashes. This matters because:

- ~**90% of a modern application is third-party code** — open-source libraries.
- These libraries get security holes (CVEs) discovered over time.
- If you don't know *exactly which version* of which library is in your software, you can't know
  if you are vulnerable.
- Governments and large organisations now **require** an SBOM for any software they buy or build.

---

## 2. The problem we are solving (NTRO)

The National Technical Research Organisation wants a tool that:
- Generates the complete SBOM of **custom-developed software** (including in-house code),
- Automatically lists the libraries / dependencies / modules used,
- **Red-flags vulnerabilities with contextual guidance** (not just "you have a problem", but
  *where, how bad, and how to fix it*),
- Supports **offline use** for sensitive/classified environments.

### What the judges will score

| Criterion | What our tool does |
|---|---|
| **Automation** | Drag a `.zip` or paste a GitHub URL → full SBOM + vulnerability report in one click. |
| **Granularity** | Lock-file based parsing → exact versions, transitive deps, integrity hashes, direct/transitive/dev flags, dependency edges. 8+ ecosystems. |
| **Accuracy** | Parses *resolved lock files* (the ground truth of what was installed), not guesses from source. |
| **Red-flag anomalies** | Bundled 836-record CVE dataset + live OSV; typosquatting, shadow deps, deprecated packages, unpinned versions, license conflicts. |
| **Context** | CVE details, CVSS score & vector, CISA Known-Exploited / malicious markers, **reachability** (is it used in your code?), exact fix command. |
| **Ease of use / UX** | Web dashboard, interactive graph, filters, one-click CycloneDX/SPDX/PDF export, plus a CLI for pipelines. |
| **Offline / in-house** | Bundled offline dataset → full scan with zero internet. |

---

## 3. How it works — the whole story (step by step)

Follow one scan from start to finish. This is the entire pipeline in plain words.

**Step 1 — You give it something to scan.**
Four ways: drag a **zip file**, type a **server folder path**, paste a **GitHub URL**, or name a
**Docker image**. The tool figures out which one you meant and starts a background job.

**Step 2 — It finds the "receipts" (lock files).**
Package managers leave behind lock files that record exactly what was installed and at which
version — e.g. `package-lock.json` (Node), `requirements.txt` / `poetry.lock` (Python),
`pom.xml` (Java), `go.mod` (Go), `Cargo.lock` (Rust), `composer.lock` (PHP), `Gemfile.lock`
(Ruby), `packages.lock.json` (.NET), `build.gradle` (Android/Java). The tool walks the whole
folder and detects **every ecosystem present at once** (polyglot support). If no lock file
exists, it falls back to manifests or source analysis.

**Step 3 — It builds the ingredient list (components).**
For each ecosystem it parses the lock file and creates a *component* for every library: its
**name**, **exact version**, **package URL (purl)**, **integrity hashes** (to verify the file
byte-for-byte), and whether it's a **direct** dependency (you asked for it) or **transitive**
(it came bundled inside something else). It also records the **dependency edges** — "A depends
on B, B depends on C" — so the full tree can be drawn.

**Step 4 — It also inspects built binaries.**
If the project contains already-built files — Python **wheels**, Java **JARs**, Windows
**`.exe`/`.dll`**, Linux **`.so`** — it extracts their metadata too, so shipped binaries land in
the SBOM even when you don't have the source.

**Step 5 — It looks inside your actual code.**
It reads the source code's **import statements** (`import requests`, `require('lodash')`,
`#include <openssl/ssl.h>`, `import com.fasterxml.jackson...`). This powers two smart features:
- **Reachability** — "is this vulnerable library *actually used* in my code, or just sitting in
  the dependency tree?"
- **Shadow dependencies** — libraries imported in code but **not declared in any manifest**
  (a red flag — could be a hidden/hijacked package).

**Step 6 — It checks every ingredient against known vulnerabilities.**
Every `name@version` is checked against:
1. a **bundled offline dataset** of 836 curated advisories across 8 ecosystems (works with no
   internet — the NTRO requirement), and
2. the **live OSV API** (Google's public vulnerability database), with a local **SQLite cache**
   so repeat scans are instant and don't hit the network.

**Step 7 — It adds context and red flags.**
For every vulnerability it attaches: the **CVE/ID**, **CVSS score** (0–10) and severity
(CRITICAL / HIGH / MEDIUM / LOW), **CISA Known-Exploited** and **malicious-package** markers,
**reachability**, and the **exact upgrade command** to fix it (e.g. `npm install lodash@4.17.21`).
It also flags anomalies: **typosquatting**, **shadow/unused dependencies**, **unpinned
versions**, **deprecated/yanked** packages, **unmaintained** packages, **license conflicts**
(copyleft code inside a commercial product), and **version drift**.

**Step 8 — It exports the official SBOM + reports.**
It emits standards-compliant **CycloneDX 1.5**, **SPDX 2.3** and **SPDX 3.0** JSON — with the
vulnerabilities, hashes, and dependency relationships **embedded inside** the file, so the SBOM
is consumable end-to-end. It also produces a self-contained **HTML report** and a **PDF report**
with an overall **Supply-Chain Health Score**.

**Step 9 — You see it in a web dashboard.**
Summary cards, an **interactive dependency graph** (vulnerable nodes glow red, attack paths
highlighted), a searchable/filterable component table, an anomaly feed, a code browser with
per-line vulnerability markers, and one-click export/download buttons.

**Step 10 — (Optional) You automate it in CI/CD.**
The same engine runs from a command line (`sbomgen`), so pipelines can scan on every commit and
**fail the build** if a critical vulnerability appears.

---

## 4. The modules explained (what & why)

### 4.1 Input handling

**Files:** `api/routes.py`, `scanner/runner.py`

- `POST /api/scan` accepts a **zip file**, a **server folder path**, a **GitHub URL**, or a
  **Docker image** (+ an `offline_only` toggle).
- Scans run in **background daemon threads**, so even huge repositories (2,000+ dependencies)
  don't block the API. The UI polls the job status until it's `complete`.
- Results are persisted to **SQLite**, so scan history survives a restart.
- Zip uploads are extracted safely with **zip-slip protection** (a malicious zip can't escape
  its folder).
- GitHub repos are fetched with a shallow `git clone`.
- Uploaded source is stored so the **code browser** (per-line vulnerability markers) keeps
  working later.

### 4.2 Ecosystem detection & lock-file parsing

**Files:** `scanner/node.py` (detection), `scanner/runner.py` (dispatch), one parser per ecosystem

The tool walks the folder, recognises lock-file names, and scans **every ecosystem found**
(parallel, not just the first).

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

**What each parser captures (highlights):**
- **npm** — handles `package-lock.json` v1/v2/v3, `yarn.lock` v1/v2, `pnpm-lock.yaml`;
  SRI integrity hashes; direct/dev flags; dependency edges.
- **Python** — `requirements.txt` (follows `-r` includes, resolves ranges via the PyPI API),
  `poetry.lock` (hashes + edges), `Pipfile.lock`, `pyproject.toml`, and `setup.py` (parsed with
  Python's AST — never executes your code).
- **Maven** — the most powerful parser: assembles the *effective POM* (inheritance, properties,
  dependencyManagement, imported BOMs), resolves the **full transitive tree** with
  nearest-wins conflict resolution, attaches licenses/homepage/hashes, and works offline too.
- **Go** — parses `require` blocks, marks `// indirect` lines as transitive, attaches **go.sum
  SHA-256 hashes**.
- **Cargo / Composer / Bundler / NuGet** — parse their lock files, capturing checksums, hashes,
  direct/dev flags and dependency edges.
- **Gradle** — manifest-only parsing plus version catalogs (`libs.versions.toml`).
  *(Known limitation: no transitive resolution for Gradle.)*

### 4.3 Binary artifact analysis

**Files:** `scanner/binary_parser.py`

Some projects ship already-built binaries without full source. The tool reads them directly:
- **Wheel** (`.whl`) → reads embedded `.dist-info/METADATA` for name/version + requirements.
- **JAR** (`.jar/.war/.ear`) → reads `META-INF/maven/*/pom.properties` + embedded `pom.xml`.
- **Windows PE** (`.exe/.dll`) → reads the **import table** to list linked DLLs (OS noise like
  `kernel32.dll` filtered out).
- **Linux ELF** (`.so/.elf`) → reads `DT_NEEDED` entries to list linked shared libraries.

Each binary gets a **SHA-256 file hash**, and its extracted dependencies are linked back as
children. (PE uses the `pefile` library; ELF parsing is hand-implemented — header → `.dynamic` →
`DT_NEEDED`.)

### 4.4 Source / import analysis (reachability, shadow & unused deps)

**Files:** `scanner/ast_scanner.py`

The tool **reads your actual code** across 10+ languages (Python via real `ast`, JS/Go/Java/C#/
Ruby/Rust/PHP via patterns, C/C++ `#include` → library mapping like `openssl/ssl.h` → openssl).
Outputs:
1. **Reachability** — a vulnerable library actually imported? Marked reachable, with the exact
   `file:line` shown.
2. **Shadow dependencies** — imported but not declared in any manifest → flagged MEDIUM and still
   CVE-scanned.
3. **Unused dependencies** — declared but never imported → flagged LOW (dead weight).

### 4.5 Vulnerability engine

**Files:** `vuln/osv.py`, `vuln/cache.py`, `vuln/seed_data.json`

Two data sources, both checked per component:
1. **Bundled offline dataset** — 836 curated advisories across 8 ecosystems (228 PyPI, 201 npm,
   120 Maven, 91 RubyGems, 69 Go, 55 Packagist, 47 crates.io, 25 NuGet). Real CVEs live in the
   `aliases` field (e.g. `GHSA-...` aliasing `CVE-2020-28500`). **This is what makes offline mode
   work.**
2. **Live OSV API** — queried with a bounded thread pool; every result cached in **SQLite** so
   repeat scans are instant and network-light.

**Version matching is precise:** advisories declare an *introduced* version (first affected,
inclusive) and a *fixed* version (safe, exclusive). If your version falls in that window, you're
vulnerable. Severity comes from **CVSS** (CRITICAL ≥ 9.0, HIGH ≥ 7.0, MEDIUM ≥ 4.0, LOW > 0),
with a fallback to advisory labels. **KEV** (CISA Known-Exploited) and **malicious** (OSV `MAL-`
prefix) markers are flagged per vulnerability.

### 4.6 Context analyzer — red flags, remediation & stats

**Files:** `context/analyzer.py`

Turns raw findings into **actionable context**:
- **Per-ecosystem fix commands:** Python → `pip install --upgrade name>=X`; Node →
  `npm install name@X`; Go → `go get name@X`; Rust → `cargo update -p name --precise X`; PHP →
  `composer require name:X`; Ruby → `bundle update name -v X`; .NET →
  `dotnet add package name --version X`; Java/Gradle → bump in the build file.
- **Anomaly checks (red flags):**
  - **Typosquatting** — name ~80% similar to a popular package (e.g. `reqests` vs `requests`).
  - **Unpinned version** — `*`, `^1.0`, `>=2.0` (you don't know what you'll get).
  - **Dependency confusion** — `file:`, `workspace:`, `git+` prefixes, or a name missing from the
    public registry (hijack risk).
  - **Unmaintained** — no release in 2+ years (HIGH if 5+ years).
  - **Suspicious provenance** — very recently published (≤90 days) and not popular.
  - **Deprecated / yanked** — PyPI yanked or npm deprecated markers.
  - **Version drift** — the same direct package at multiple versions.
  - **Copyleft license conflict** — AGPL/SSPL (HIGH), GPL (MEDIUM) inside a permissive project.
- Final **stats** power the dashboard: total/direct/transitive components, vulnerable
  components, unique vulnerabilities, per-severity counts, max severity, KEV count, malicious
  count, reachable vulns, binary/lockfile/import counts, anomaly count.

### 4.7 SBOM generation — CycloneDX & SPDX

**Files:** `sbom/generator.py`, `sbom/spdx30.py`, `sbom/hashing.py`

Emits three industry-standard formats:
- **CycloneDX 1.5** (JSON) — built with the **official `cyclonedx-python-lib`** (never a
  hand-rolled schema), with purls, hashes, licenses, scope, custom properties, and **embedded
  `Vulnerability` objects** (CVSS rating, "Fixed in X" recommendation, KEV/malicious/reachable
  flags).
- **SPDX 2.3** (JSON) — one `SPDXRef-Package` per component with checksums, `externalRefs`,
  `DEPENDS_ON` edges, vulnerabilities linked via `AFFECTS`, an NTIA-minimum root package, and
  every project file `CONTAINS`-ed with SHA-256 hashes.
- **SPDX 3.0** (JSON) — next-gen format with Package/Vulnerability/Relationship elements.

Key point: **the SBOMs aren't just name lists** — vulnerabilities, integrity hashes and
dependency relationships are embedded *inside* the file, so it's consumable end-to-end.
`hashing.py` computes SHA-256 + SHA-1 for every project file (capped at 2 MiB / 20,000 files).

### 4.8 Reports — HTML & PDF

**Files:** `report/generator.py`, `report/pdf.py`

Executive-friendly outputs: a **Supply-Chain Health Score** circle, severity bar, stat cards,
vulnerability findings table (with CVE links, flag badges, "Used In file:line", "Fixed In",
remediation command), anomaly table, full component inventory, binaries, and license summary.
**Health score formula:** `100 − 30×(critical) − 12×(high) − 5×(medium) − 2×(low)`, clamped 0–100.

### 4.9 Policy engine & CLI — CI/CD gating

**Files:** `policy/engine.py`, `backend/sbomgen.py`

- **Policy engine** lets an admin define rules: `max_severity`, `max_vulnerabilities`,
  `fail_on_kev`, `fail_on_malicious`, blocked packages, blocked/allowed licenses. Every scan gets
  a `passed`/`failed` verdict with violations. Default policy blocks CRITICAL, any vulnerability
  count > 0, KEV, malicious, and AGPL/SSPL licenses.
- **CLI:** `python -m sbomgen scan <folder|zip|github-url> --offline --format cyclonedx -o sbom.json`.
  Formats: `json`, `cyclonedx`, `spdx`, `spdx30`, `html`, `pdf`. **`--fail-on <severity>`** exits
  with code **2** when the worst vulnerability crosses the threshold — so the CI build fails.
  Exit codes: 0 = success, 1 = scan/IO error, 2 = gate tripped.

### 4.10 Auth & multi-tenancy

**Files:** `auth/` , `api/auth_routes.py`

Multiple organisations with real accounts; users get roles (admin / analyst / viewer); admins
manage users and generate API tokens; every important action is written to an **audit log**.
Passwords bcrypt-hashed; login uses JWTs (12h expiry); orgs only see their own scans.

### 4.11 Frontend dashboard

**Files:** `frontend/src/App.jsx`, `graph.js`, `components/*`

- **Summary cards** — totals, vulnerable counts, reachable vulns, KEV count, anomalies, health
  score ring, severity bar.
- **Dependency graph** — interactive tree (d3-hierarchy). Vulnerable nodes glow red; attack
  paths from root to a vulnerable node highlighted; import/binary nodes get dashed rings;
  zoom/pan/expand/collapse.
- **Components table** — searchable + filterable by severity, ecosystem, type
  (direct/transitive/dev), vulnerable-only, and origin (lockfile/manifest/import/binary).
- **Context panel** — click any component: used-in-code locations, parent/child relationships,
  integrity hashes, and each vulnerability with CVSS, KEV/malicious/reachable badges, and a
  copy-paste **fix command**.
- **Code browser** — browse the uploaded source; vulnerable lines marked with severity colors.
- **Anomalies / Licenses / Binaries tabs** — red-flag feed, license compliance view with copyleft
  warnings, extracted binary artifacts.
- **Scan history** — compare (diff) two scans (added/removed components, new/resolved
  vulnerabilities, before/after score) and merge into a combined SBOM.

The built frontend is served directly by FastAPI (single-binary deployment), and a PyInstaller
build launches a desktop window (pywebview/Edge) with export-to-Downloads.

---

## 5. Key design decisions & honest limitations

**Why these choices make it strong:**

| Decision | Why |
|---|---|
| **Lock files over source scanning** | Lock files record *exact resolved versions* — the ground truth. No guessing. |
| **Polyglot in one pass** | One scan covers mixed projects (Python + JS + Java...) automatically. |
| **Binary artifacts counted** | Wheels/JARs/PE/ELF parsed directly — SBOM even without source. |
| **Lock-file-less fallback** | AST/import analysis still surfaces the packages actually used. |
| **Bundled dataset + live OSV + SQLite cache** | Offline-safe for NTRO, fresh online, fast repeat scans. |
| **Official CycloneDX library** | Standards compliance without schema errors. |
| **Vulnerabilities embedded in SBOMs** | One consumable file carries the whole picture. |
| **Source-aware analysis** | Reachability/shadow/unused deps turn a list into actionable risk. |
| **Background jobs + polling** | 2,000+ dependency repos don't block the API. |
| **Policy + CLI** | It's not just a UI — it's a CI/CD guard. |
| **Everything in SQLite** | No DB server; portable; restart-safe. |

**Known limitations (be honest if asked):**
- Gradle is manifest-only (no transitive resolution).
- Yarn lock parsing gives a flat list (no dependency edges); directness comes from
  `package.json`.
- KEV detection is a heuristic from advisory metadata, not a live CISA feed.
- OSV lookups are one request per component (8 concurrent threads), not a single batch call.
- Semver range matching ignores pre-release/build suffixes.

---

## 6. Tech stack at a glance

| Layer | Technology | Why |
|---|---|---|
| Backend API | **Python · FastAPI** | Fast REST API, Pydantic validation, background threading |
| Frontend | **React + Vite** | Single-page dashboard |
| Dependency graph | **d3-hierarchy** (custom SVG) | Interactive tree / attack-path visualization |
| CycloneDX output | **cyclonedx-python-lib** | Official, standards-compliant library |
| Persistence | **SQLite** | Jobs, OSV cache, Maven cache, auth, audit log — no DB server |
| Vulnerability data | **OSV API** (live) + **bundled seed_data.json** (offline) | Freshness + air-gapped operation |
| PDF reports | **reportlab** | A4 print-ready reports |
| Packaging | **PyInstaller** | Single `.exe` — no installs needed |
| Auth | **JWT + bcrypt** | Multi-tenant orgs, roles, API tokens, audit log |

---

## 7. Quick answers to top judge questions

**What is an SBOM and why does it matter?**
It's the machine-readable ingredients list of software — every component with its version and
hash. It matters because ~90% of modern software is third-party code, and you can't secure what
you don't know you have. Regulators and agencies (like NTRO) now require one for procurement.

**Why lock files instead of scanning the source?**
Lock files record the *exact resolved* versions actually installed, including every transitive
dependency — the ground truth. Source code only shows version ranges (e.g. `^4.0.0`), which can
resolve to many different versions.

**What if there's no lock file?**
We fall back: (1) parse the manifest and resolve version ranges via the registry API, (2)
analyse the actual imports in the code. Accuracy drops slightly, but we still surface the
third-party packages actually used — and flag unpinned versions as an anomaly.

**How does offline mode work?**
All vulnerability data ships inside the tool — 836 curated advisories across 8 ecosystems, plus
offline license metadata. Version matching, CVSS scoring and fix commands all work from this
local data. Online mode layers the live OSV API on top, cached in SQLite.

**How is this different from `npm audit` / Dependabot?**
Those are single-ecosystem vulnerability lists. Ours is a unified, polyglot tool that also
parses binaries, emits standard SBOMs (CycloneDX/SPDX), adds reachability, shadow/typosquat/
license checks, an offline mode, an executive health-score report, and a CI/CD policy gate.

**What does "reachable" mean?**
We parse the actual import statements in the code. If a vulnerable package is imported, we mark
it reachable and show the exact `file:line`; otherwise we still report it but flag it as lower
priority.

**How are licenses handled?**
Each component's license is captured and classified permissive vs copyleft. Copyleft
(AGPL/SSPL/GPL) is flagged, with a HIGH "license conflict" when a permissive project pulls in
copyleft code. A license dashboard shows the breakdown.

**Can it handle compiled / binary-only software?**
Yes — wheels, JARs, Windows PE files and Linux ELF files are parsed directly (embedded
metadata / import table / DT_NEEDED) and produce an SBOM with file hashes even without source.

**How does CI/CD gating work?**
The CLI's `--fail-on <severity>` exits 2 when the worst vulnerability crosses the threshold,
and the policy engine enforces rules (max severity, KEV, malicious, blocked packages/licenses).
Your pipeline can block the merge, not just warn.

**How accurate is the version matching?**
Advisories declare introduced (inclusive) / fixed (exclusive) ranges; we compare your exact
version from the lock file against that window numerically. Parsers handle real formats (npm
v1–v3, yarn v1/v2, go.sum hashes, etc.) and were verified against real npm projects.

---

## 8. How to run it (quick)

```powershell
# Backend
cd backend
pip install -r requirements.txt
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000

# Frontend (dev, hot reload)
cd frontend
npm install
npm run dev            # http://localhost:5173 (proxies /api → :8000)

# Or build once and FastAPI serves it at http://127.0.0.1:8000/
cd frontend && npm run build

# CLI / CI-CD
python -m sbomgen scan D:\repos\my-app --offline --format cyclonedx -o sbom.json
python -m sbomgen scan app.zip --format json
python -m sbomgen scan https://github.com/owner/repo --format pdf -o report.pdf
python -m sbomgen scan D:\repos\my-app --fail-on high    # exit 2 if anything ≥ HIGH
```

---

## 9. Glossary

| Term | Easy meaning |
|---|---|
| **SBOM** | Software Bill of Materials — machine-readable ingredient list of software. |
| **Dependency** | A library your project uses. |
| **Direct dependency** | A library you declared/asked for yourself. |
| **Transitive dependency** | A library that arrived inside another library. |
| **Lock file** | File recording the *exact* versions installed (`package-lock.json`, `Cargo.lock`...). |
| **Manifest** | File where you *declare* dependencies, usually with version ranges. |
| **Component** | One library/package in the SBOM. |
| **purl** | Package URL — universal package name, e.g. `pkg:npm/lodash@4.17.12`. |
| **Integrity hash** | Cryptographic fingerprint (SHA-1/256/512) to verify a file is untampered. |
| **CVE** | Common Vulnerabilities and Exposures — public ID for a known flaw. |
| **CVSS** | Vulnerability scoring system — 0–10 danger score plus an explaining vector. |
| **OSV** | Open Source Vulnerabilities — Google's public vulnerability database (API). |
| **KEV** | CISA Known Exploited Vulnerabilities — flaws actually being exploited in the wild. |
| **Reachability** | Whether a vulnerable package is actually imported/used in your code. |
| **Shadow dependency** | Imported in code but not declared in any manifest — red flag. |
| **Typosquatting** | Registering a package with a name nearly identical to a popular one. |
| **CycloneDX / SPDX** | The two major SBOM standards (JSON). |
| **Attack path** | Chain of dependencies from your root project down to a vulnerable package. |

---

*Medium-length overview for SIH 2024 (Problem 1449). For the full technical deep-dive, demo
script, all run instructions and expanded Q&A, see `DOCUMENTATION.md`.*
