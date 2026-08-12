# SBOM-Gen — Developer's Guide

**SIH 2024 · Problem Statement 1449 · NTRO (National Technical Research Organisation)**

> This document is written the way I (the developer) talk about the project — plain words,
> no over-explaining. Read it once and you'll understand what we built, how it works end to
> end, and which technology is doing what. That's also everything you need to answer judges.

---

## 1. What we built (one line)

SBOM-Gen is a tool that takes any software project (a zip, a folder, a GitHub link) and
automatically produces its **Software Bill of Materials (SBOM)** — the machine-readable
"ingredients list" of every third-party library it uses — checks those libraries against known
**vulnerabilities**, tells you **exactly how to fix them**, and exports everything in the
government-standard formats, all while working **fully offline**.

### Why does this matter? (the 30-second story)

- ~80–90% of a modern application is not your code — it's open-source libraries.
- These libraries get security holes (CVEs) discovered over time.
- If you don't know *exactly which version* of *which library* is inside your software, you
  can't know if you're vulnerable.
- Governments and agencies now **require** an SBOM for software they buy or build.
- NTRO added one extra twist: the tool must work **offline**, because classified environments
  can't reach public vulnerability databases over the internet.

So our job: **automatically list everything, red-flag what's dangerous, show how to fix it,
and make the official SBOM file — all offline.**

---

## 2. The big picture — how one scan flows

Think of the whole tool as an assembly line. One scan goes through these stages, top to bottom:

```
┌─────────────────────────────────────────────────────────────┐
│  REACT DASHBOARD  (frontend/, Vite)                          │
│  Upload zip / folder / GitHub URL  →  shows results visually │
└──────────────────────────┬──────────────────────────────────┘
                           │ REST API calls (JSON)
┌──────────────────────────▼──────────────────────────────────┐
│  FASTAPI BACKEND  (backend/, Python)                         │
│  POST /api/scan  → starts a background job → returns job id  │
│  GET  /api/scan/{id} → poll status + result                  │
└──────────────────────────┬──────────────────────────────────┘
     ┌─────────────┬───────┴─────────┬───────────────┬──────────┐
     ▼             ▼                 ▼               ▼          ▼
 ┌──────────┐ ┌──────────┐   ┌───────────┐   ┌───────────┐  ┌───────────┐
 │ detect   │ │ parsers  │   │ vuln/     │   │ context/  │  │ sbom/     │
 │ which    │ │ read     │   │ osv.py    │   │ analyzer  │  │ generator │
 │ package  │ │ lock     │   │ seed_data │   │ CVSS, fix │  │ CycloneDX │
 │ managers │ │ files    │   │ offline   │   │ command,  │  │ + SPDX    │
 │ are used │ │ → 18+    │   │ + live OSV│   │ anomalies │  │ 2.3 / 3.0 │
 │          │ │ formats  │   │ SQLite    │   │ reachable │  │           │
 └──────────┘ └──────────┘   └───────────┘   └───────────┘  └───────────┘
```

In plain words, the same pipeline is:

1. **You give it something to scan** — a zip file, a server folder path, or a GitHub URL.
2. **It finds the "receipts"** — lock files like `package-lock.json`, `requirements.txt`,
   `pom.xml`, `Cargo.lock` that record exactly what was installed and at which version.
3. **It builds the ingredient list** — every library becomes a *component* with name, exact
   version, package URL (purl), integrity hashes, and whether it's a direct dependency (you
   asked for it) or transitive (it came inside something else).
4. **It also reads compiled binaries** — wheels, JARs, `.exe`/`.dll`, `.so` — and pulls out
   their metadata, so shipped artifacts land in the SBOM even with no source.
5. **It looks inside your actual code** — reads import statements to decide *reachability*
   (is the vulnerable library actually used?) and find *shadow dependencies* (imported but
   not declared anywhere).
6. **It checks every ingredient against vulnerabilities** — a bundled offline dataset + the
   live OSV API, cached in SQLite.
7. **It adds context and red flags** — CVE ID, CVSS score, severity, CISA Known-Exploited /
   malicious markers, reachability, and the exact upgrade command.
8. **It generates the official SBOMs** — CycloneDX 1.5 and SPDX 2.3/3.0 JSON, with the
   vulnerabilities and hashes embedded inside the file.
9. **You see it in the dashboard** — summary cards, an interactive dependency graph (vulnerable
   nodes glow red), a searchable table, an anomaly feed, and one-click exports (CycloneDX,
   SPDX, HTML report, PDF report).

---

## 3. The tech stack — what each piece does

| Layer | Technology | What it actually does for us |
|---|---|---|
| Backend API | **Python + FastAPI** | The server. Receives scan requests, runs them in background threads, serves results as JSON. |
| Data models | **Pydantic** | Defines the `Component`, `Vulnerability`, `Anomaly`, `ScanResult` objects — typed and validated. `models.py`. |
| Frontend | **React + Vite** | The web dashboard users interact with. Vite is the build tool that bundles it into static files. |
| Dependency graph | **d3-hierarchy** | Draws the interactive dependency tree. Red nodes = vulnerable. No force-layout library — we build a hierarchy tree manually. |
| CycloneDX output | **cyclonedx-python-lib** | The *official* library for generating standards-compliant CycloneDX JSON — we never hand-roll the schema. |
| SPDX output | hand-built JSON | We build SPDX 2.3 / 3.0 ourselves so we can embed vulnerabilities, file hashes, and relationships exactly how we want. |
| Persistence | **SQLite** | One file database. Stores scan history, the OSV API cache, Maven resolution cache, user accounts, audit log. No DB server to install. |
| Vulnerability data | **OSV API (live) + bundled `seed_data.json` (offline)** | OSV is Google's public vulnerability database. The bundled file has ~836 curated advisories across 8 ecosystems — this is what makes offline mode work. |
| PDF reports | **reportlab** | Generates the A4 printable executive PDF report. |
| Packaging | **PyInstaller** | Bundles everything (Python + built React app) into a single `.exe` — no installs needed. |
| Desktop shell | **pywebview** | Wraps the web app in a native window (EdgeChromium) so double-clicking the exe opens the app like a desktop program. |
| Auth | **JWT + bcrypt** | Multi-organisation accounts, roles (admin/analyst/viewer), API tokens, audit log. |

### The two clever bits worth remembering

- **Offline mode:** all the CVE data ships *inside* the tool (the `seed_data.json`), so version
  matching, severity scoring, and fix commands all work with zero internet. This is the NTRO
  requirement.
- **Standards, not guesses:** CycloneDX uses the official library; SPDX is hand-built but
  deterministic. Both embed the vulnerabilities and hashes *inside* the SBOM file, so the file
  alone tells the whole security story.

---

## 4. How the scan works, module by module

All backend code lives under `backend/app/`. This is the part to know cold.

### 4.1 Input handling — `api/routes.py` + `scanner/runner.py`

The API accepts the input and kicks off a background job:

- `POST /api/scan` accepts a **zip file**, a **folder path**, a **GitHub URL** (and Docker
  image names). It figures out which one you meant and returns a `job_id` immediately.
- The scan runs in a **daemon thread**, so a repo with 2,000+ dependencies doesn't block the
  server. The UI polls `GET /api/scan/{id}` every 1.5 s until the status is `complete`.
- Zip uploads are extracted with **zip-slip protection** — it resolves every member path and
  refuses anything that would escape the destination folder.
- GitHub repos are fetched with a shallow `git clone --depth 1`.
- Results are stored in **SQLite** (`jobs.db`), so scan history survives a restart.
- The uploaded source is persisted too, so the **code browser** in the UI keeps working later.

### 4.2 Ecosystem detection & lock-file parsing — `scanner/`

This is the heart of the tool. First `node.detect()` walks the folder and recognises lock-file
names. Whatever it finds tells us which package managers the project uses — a project can use
several at once (a Python backend + a JS frontend), and we scan **all of them in parallel**.

Then the dispatcher (`runner.py`) picks the *most authoritative* file per ecosystem (lock files
over plain manifests, because lock files have exact resolved versions). There's one parser per
ecosystem:

| File | Ecosystem |
|---|---|
| `node.py` | npm — `package-lock.json` (v1/v2/v3), `yarn.lock` (v1/v2), `pnpm-lock.yaml`, `package.json` |
| `python_parser.py` | pypi — `requirements.txt`, `poetry.lock`, `Pipfile.lock`, `pyproject.toml`, `setup.py` |
| `maven_resolver.py` + `java_parser.py` | Java Maven — `pom.xml` (resolves the full transitive tree) |
| `go_parser.py` | Go — `go.mod` + `go.sum` (integrity hashes) |
| `cargo_parser.py` | Rust — `Cargo.lock` |
| `composer_parser.py` | PHP — `composer.lock` |
| `bundler_parser.py` | Ruby — `Gemfile.lock` |
| `nuget_parser.py` | .NET — `packages.lock.json`, `project.assets.json` |
| `gradle_parser.py` | Gradle — `build.gradle(.kts)` + version catalogs |
| `podfile_parser.py` | iOS — `Podfile.lock` |
| `conan_parser.py`, `conda_parser.py`, `vcpkg_parser.py`, `linux_parser.py` | C/C++ and OS packages |
| `docker_parser.py` | Docker images (OS packages + lock files inside) |

What every parser captures per library: **name**, **exact version**, **purl**, **hashes**,
**direct/transitive/dev flags**, and **dependency edges** (who depends on whom). Two parsers
worth bragging about:

- **Maven** is the most powerful — it assembles the *effective POM* (inheritance, `${}`
  properties, `dependencyManagement`, imported BOMs), resolves the **full transitive tree**
  with nearest-wins conflict resolution, and can fetch parent POMs from Maven Central
  (SQLite-cached) or use a bundled offline tree.
- **Python** parses `setup.py` with Python's own **AST parser** — it reads the code structure
  but *never executes* your setup script (that would be a security risk).

### 4.3 Binary artifacts — `scanner/binary_parser.py`

Some projects ship already-built binaries without full source. We read those directly:

- **Python wheel** (`.whl`) → reads the embedded `.dist-info/METADATA` for name/version + deps.
- **Java JAR** (`.jar`) → reads `META-INF/maven/*/pom.properties` + embedded `pom.xml`.
- **Windows PE** (`.exe/.dll`) → reads the **import table** to list linked DLLs (filtering OS
  noise like `kernel32.dll`). Uses the `pefile` library.
- **Linux ELF** (`.so/.elf`) → reads `DT_NEEDED` entries for linked shared libraries
  (hand-implemented parser).

Each binary gets a **SHA-256 file hash**, and its extracted dependencies are linked back as
children.

### 4.4 Source / import analysis — `scanner/ast_scanner.py`

We read the *actual code* to figure out what's really used. Python uses the real `ast` module;
other languages (JS, Go, Java, C#, Ruby, Rust, PHP) use careful patterns; C/C++ maps
`#include <openssl/ssl.h>` → `openssl`. Three outputs:

1. **Reachability** — if a vulnerable library is in the tree but your code never imports it,
   the risk is lower. We mark it *reachable* or not, with the exact `file:line`.
2. **Shadow dependencies** — imported in code but not declared in any manifest. A red flag
   (could be a hijacked package) — flagged MEDIUM and still CVE-scanned.
3. **Unused dependencies** — declared but never imported — flagged LOW.

### 4.5 Vulnerability engine — `vuln/osv.py` + `vuln/seed_data.json`

Every `name@version` is checked against two sources:

1. **Bundled offline dataset** (`seed_data.json`) — ~836 curated advisories across 8 ecosystems
   (npm, PyPI, Maven, RubyGems, Go, Packagist, crates.io, NuGet). **This is what makes offline
   mode work.** Real CVE numbers live in the `aliases` field (e.g. a `GHSA-...` advisories a
   `CVE-2020-28500`).
2. **Live OSV API** — Google's public database, queried at scan time with an 8-thread pool.
   Every result is cached in **SQLite**, so repeat scans are instant and network-light.

**Version matching is precise:** each advisory declares an *introduced* version (first affected,
inclusive) and a *fixed* version (safe, exclusive). If your version sits in that window, you're
vulnerable. Severity comes from **CVSS** (CRITICAL ≥ 9.0, HIGH ≥ 7.0, MEDIUM ≥ 4.0, LOW > 0),
with a fallback to GitHub/npm labels when no CVSS exists. **KEV** (CISA Known-Exploited) and
**malicious** (`MAL-` prefix) flags are attached per vulnerability.

### 4.6 Context analyzer — `context/analyzer.py`

Knowing "you have a vulnerability" is only half the job. This module adds the actionable part:

- **Fix command** per ecosystem: Python → `pip install --upgrade requests>=2.32.0`,
  Node → `npm install lodash@4.17.21`, Go → `go get ...@v1.2.3`, Rust →
  `cargo update -p serde --precise 1.0.200`, etc.
- **Red-flag anomalies** it hunts for:
  - **Typosquatting** — name ~80% similar to a popular package (`reqests` vs `requests`).
  - **Unpinned version** — `*`, `^1.0`, `>=2.0` (you don't know what you'll get).
  - **Dependency confusion** — `file:`/`workspace:`/`git+` prefixes or a name missing from the
    public registry.
  - **Unmaintained** — no release in 2+ years (HIGH if 5+).
  - **Suspicious provenance** — published ≤90 days and not popular.
  - **Deprecated / yanked** packages.
  - **Version drift** — same direct package at multiple versions.
  - **Copyleft license conflict** — AGPL/SSPL/GPL inside a permissive project.
- Final **stats** power the dashboard cards (totals, vulnerable counts, health score, etc.).

### 4.7 SBOM generation — `sbom/generator.py`, `sbom/spdx30.py`

Emits the official files:

- **CycloneDX 1.5** — built with the official `cyclonedx-python-lib` (never hand-rolled).
  Includes purls, hashes, licenses, direct/dev scope, a root dependency linking every direct
  component, and **embedded `Vulnerability` objects** (CVSS rating, "Fixed in X" recommendation,
  KEV/malicious/reachable flags).
- **SPDX 2.3** — one package element per component with checksums, `externalRefs` (purl +
  license), `DEPENDS_ON` relationships, vulnerabilities linked via `AFFECTS`, and every project
  file `CONTAINS`-ed with SHA-256 hashes.
- **SPDX 3.0** — next-gen JSON-LD format.

`hashing.py` computes SHA-256 + SHA-1 for every project file (capped at 2 MiB/file and 20,000
files to stay fast).

### 4.8 Reports — `report/generator.py`, `report/pdf.py`

Executive-friendly outputs: a **Supply-Chain Health Score** circle, severity bar, stat cards,
a vulnerability findings table (with CVE links, flag badges, "Used In file:line", "Fixed In",
remediation command), anomaly table, full component inventory, and license summary.

**Health score formula:** `100 − 30×(critical) − 12×(high) − 5×(medium) − 2×(low)`, clamped 0–100.

### 4.9 Policy + CLI — `policy/engine.py`, `backend/sbomgen.py`

Two ways the tool gates a pipeline:

- **Policy engine:** admin rules like `max_severity`, `max_vulnerabilities`, `fail_on_kev`,
  `fail_on_malicious`, blocked packages, blocked/allowed licenses. Each scan gets a
  `passed`/`failed` verdict. Default policy blocks CRITICAL, any vulns, KEV, malicious, and
  AGPL/SSPL licenses.
- **CLI:** `python -m sbomgen scan <folder|zip|github-url> --format cyclonedx -o sbom.json`.
  `--fail-on <severity>` **exits with code 2** when the worst vulnerability crosses the
  threshold — so the CI build fails. Exit codes: 0 = success, 1 = scan error, 2 = gate tripped.

### 4.10 Auth — `auth/`

Multi-organisation accounts. Passwords are bcrypt-hashed, login uses JWTs (12-hour expiry),
users get roles (admin/analyst/viewer), admins manage users and API tokens, and every important
action is written to an audit log. Each org only sees its own scans.

### 4.11 Frontend dashboard — `frontend/src/`

The UI is a React single-page app with these views:

- **Summary cards** — totals, vulnerable counts, reachable vulns, KEV count, anomalies, an
  animated health-score ring, severity bar.
- **Dependency graph** — interactive tree built with `d3-hierarchy`. Vulnerable nodes glow red,
  attack paths from root to a vulnerable node are highlighted, zoom/pan/expand/collapse.
- **Components table** — searchable + filterable by severity, ecosystem, type, and
  vulnerable-only.
- **Context panel** — click any component: used-in-code locations, parent/child relationships,
  hashes, and each vulnerability with CVSS + the copy-paste fix command.
- **Code browser** — browse the uploaded source; vulnerable lines marked with severity colors.
- **Anomalies / Licenses / Binaries tabs** — red-flag feed, license compliance, extracted
  binaries.
- **Scan history** — compare (diff) two scans: added/removed components, new/resolved
  vulnerabilities, before/after score.

The built frontend (`frontend/dist`) is served directly by FastAPI, so one server serves both
the API and the UI. In dev, Vite runs on port 5173 and proxies `/api` → `:8000`.

---

## 5. The desktop app — how the exe works

`SBOM-Gen.spec` is the PyInstaller spec. The build bundles:

- the Python backend + FastAPI + all libraries,
- the pre-built React frontend (`frontend/dist`),
- the offline CVE dataset,
- and an icon.

When you double-click `SBOM-Gen.exe` (`backend/app/main.py` → `run()`):
1. It starts a FastAPI server on a **free local port** (or `SBOM_PORT` if set).
2. It waits for the health check.
3. It opens a native window using **pywebview** (EdgeChromium), with a JS bridge that saves
   exported SBOMs and PDFs straight to `~/Downloads`.
4. If the native window isn't available, it falls back to opening the default browser.
5. Logs go to `%LOCALAPPDATA%\sih1449\logs\app.log`.

That's how we get a "double-click and it just works" desktop tool with no Python/Node installs.

---

## 6. How to run it

**Backend only (API + UI):**
```powershell
cd backend
pip install -r requirements.txt
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000
# open http://127.0.0.1:8000
```

**Frontend dev mode (hot reload):**
```powershell
cd frontend
npm install
npm run dev          # http://localhost:5173 (proxies /api → :8000)
```

**CLI / CI-CD:**
```powershell
cd backend
python -m sbomgen scan D:\repos\my-app --offline --format cyclonedx -o sbom.json
python -m sbomgen scan app.zip --format json
python -m sbomgen scan https://github.com/owner/repo --format pdf -o report.pdf
python -m sbomgen scan D:\repos\my-app --fail-on high    # exit 2 if anything >= HIGH
```

**Standalone exe:**
```powershell
cd frontend && npm run build
cd ..
python -m PyInstaller SBOM-Gen.spec --noconfirm   # -> dist\SBOM-Gen.exe
```

**Tests:**
```powershell
cd backend
python tests\test_e2e.py          # end-to-end API tests
python tests\test_regression.py   # severity/robustness
python tests\test_binary.py       # binary artifact parsing
python tests\test_cli.py          # CLI exit codes and formats
```

---

## 7. Repository map (where things live)

```
backend/
  app/
    main.py               FastAPI entrypoint + desktop run() (pywebview)
    models.py             Pydantic models (Component, Vulnerability, ScanResult...)
    api/routes.py         REST endpoints + background jobs
    api/auth_routes.py    login, users, tokens, audit log
    scanner/              detection + per-ecosystem parsers + binary + AST
    vuln/                 osv.py (live), cache.py (SQLite), seed_data.json (offline)
    context/analyzer.py   severity, remediation, reachability, anomalies, stats
    sbom/                 generator.py (CycloneDX + SPDX), spdx30.py, hashing.py
    report/               generator.py (HTML), pdf.py (reportlab)
    policy/               engine.py (CI/CD rules)
    auth/                 security.py, store.py, deps.py
  sbomgen.py              the CLI
  tests/                  e2e, regression, binary, CLI tests
  scripts/build_seed.py   rebuilds the offline CVE dataset (build-time)
frontend/
  src/App.jsx             layout, state, polling
  src/api.js              API client + scan polling
  src/graph.js            dependency-tree builder from a scan result
  src/components/         Uploader, SummaryCards, DependencyGraph, ComponentTable,
                          ContextPanel, AnomalyList, CodeBrowser, ScanHistory, ...
samples/                  demo projects (npm-app, py-app, multi-app polyglot)
```

---

## 8. Quick answers to the questions judges always ask

**What is an SBOM?**
The machine-readable "ingredients list" of software — every component with its version and
integrity hash. It matters because ~90% of modern software is third-party code, and you can't
secure what you don't know you have.

**Why lock files and not source scanning?**
Lock files record the *exact resolved* versions actually installed, including every transitive
dependency — the ground truth. Source only shows version ranges (`^4.0.0`), which could resolve
to many different versions.

**How does offline mode work?**
All vulnerability data ships inside the tool — ~836 curated advisories across 8 ecosystems.
Matching, severity, and fix commands all work from this local data. Online mode layers the live
OSV API on top, cached in SQLite.

**What makes this different from `npm audit` / Dependabot?**
Those are single-ecosystem vulnerability lists. Ours is a unified, polyglot tool that also
parses binaries, emits standard SBOMs (CycloneDX/SPDX), adds reachability, typosquatting,
license checks, an offline mode, executive reports, and a CI/CD policy gate.

**How accurate is the version matching?**
Advisories declare an introduced (inclusive) / fixed (exclusive) window; we compare your exact
version from the lock file against that window numerically. Real formats verified against real
projects.

---

*Full technical deep-dive: `DOCUMENTATION.md` · Plain-English overview: `PROJECT_OVERVIEW.md` ·
Run + demo script: `README.md`*











































# Tech Stack & Libraries

SBOM-Gen is a polyglot SBOM generation and vulnerability scanning tool. It combines a
Python/FastAPI backend with a React dashboard, packaged as a single desktop exe.

---

## 1. Backend — Python + FastAPI

The core scanning engine and REST API live in `backend/`.

| Library | Purpose |
|---|---|
| **FastAPI** | REST API framework — `/api/scan`, polling, CycloneDX/SPDX/PDF export endpoints. Ships with **Pydantic** for the shared `Component`, `Vulnerability`, `ScanResult` models. |
| **Uvicorn** | ASGI server that runs the FastAPI app (dev, and embedded inside the desktop exe). |
| **python-multipart** | Parses the drag-and-drop zip file uploads. |
| **requests** | HTTP client for the live **OSV API** (vulnerability lookups), PyPI/npm registry resolution, and ClearlyDefined license enrichment. |

### Ecosystem parsers
Most lock-file parsing (`package-lock.json`, `yarn.lock`, `Cargo.lock`, `composer.lock`,
`Gemfile.lock`, `packages.lock.json`, `pom.xml`, `go.mod`+`go.sum`, Gradle, Conan, vcpkg, conda,
Podfile, dpkg) is done with **pure Python stdlib** (`json`, `re`, `ast`, `xml.etree`) — no heavy
third-party dependencies.

| Library | Purpose |
|---|---|
| **cyclonedx-python-lib** | Official library for standards-compliant **CycloneDX 1.5** output — components, purls, hashes, licenses, dependency relationships and embedded vulnerabilities. Includes `packageurl` for package-URL generation. |
| **pefile** | Reads Windows **PE** exe/dll import tables to list linked DLLs in binary artifacts. |
| — | Linux **ELF** `DT_NEEDED` parsing and wheel/JAR metadata extraction are hand-implemented. |
| **reportlab** | Generates the A4 **PDF** executive report. |
| **packaging** | Semver/version-range parsing used for vulnerability range matching. |
| **PyJWT** | JSON Web Tokens for login sessions (multi-tenant orgs, roles, API tokens). |
| **bcrypt** | Password hashing for the auth store. |
| **cryptography** | Ed25519 signing/verification of SBOM manifests. |
| **pywebview** | Opens the packaged exe as a native desktop window (Edge Chromium backend) with a bridge to export files to Downloads. |
| **sqlite3** (stdlib) | Persistence for scan jobs/history, the OSV result cache, Maven metadata cache, auth, policy and audit log — no database server needed. |
| **cose** | Optional COSE verification for Microsoft-style signed catalogs. |

### Vulnerability data
- **Bundled offline dataset** — `seed_data.json` with 830+ curated advisories across 8
  ecosystems (this is what makes fully offline scans work).
- **Live OSV API** — checked per component through a bounded thread pool, cached in SQLite.

### Standard-library highlights
`ast` (Python import analysis for reachability), `hashlib` (SHA-1/256 file hashes),
`concurrent.futures` + `threading` (parallel hashing/parsing, background scan jobs), `zipfile`
(zip extraction with zip-slip protection), `subprocess` (shallow `git clone` for GitHub URLs,
docker CLI fallback), `ctypes`/`winreg` (GPU detection + registry tweaks on Windows).

---

## 2. Frontend — React + Vite

The web dashboard lives in `frontend/` and is served directly by FastAPI after `npm run build`.

| Library | Purpose |
|---|---|
| **React 18** + **react-dom** | UI framework — SPA dashboard. |
| **Vite 5** | Build tool and dev server (hot reload in dev, static bundle for production). |
| **@vitejs/plugin-react** | React JSX support in Vite. |
| **d3-hierarchy** | Renders the interactive **dependency graph** — vulnerable nodes glow red, attack paths highlighted, zoom/pan/collapse. |
| **lucide-react** | Icon set for the dashboard UI. |
| **@fontsource/inter**, **@fontsource/jetbrains-mono** | Inter UI font + JetBrains Mono for code/hash display. |

### Dev-only
| Library | Purpose |
|---|---|
| **puppeteer-core** | UI check/screenshot automation (development only, not shipped). |

---

## 3. CLI & Packaging

| Tool | Purpose |
|---|---|
| **`sbomgen` CLI** | Python `argparse`-based command-line tool (`backend/sbomgen.py`). Scans folders/zips/GitHub URLs, emits `json`, `cyclonedx`, `spdx`, `spdx30`, `html`, `pdf`, and gates CI with `--fail-on <severity>` (exit code 2 = build failure). |
| **PyInstaller** | Builds the single-file executables — `SBOM-Gen.spec` (windowed desktop app with icon) and `SBOM-Gen-CLI.spec` (console CLI). |
| **Inno Setup** | `installer.iss` — Windows installer that bundles the built exe. |

---

## 4. Testing & Tooling

| Tool | Purpose |
|---|---|
| **pytest** + FastAPI **TestClient** | E2E API tests, severity/regression tests, binary-artifact tests, CLI subprocess tests (`backend/tests/`). |
| **pip** (`requirements.txt`) / **npm** (`package-lock.json`) | Dependency management for backend and frontend. |
| **uvicorn / vite** | Local dev loop: backend on `:8000`, frontend on `:5173` (proxying `/api`). |

---

## 5. Tech Stack at a Glance

| Layer | Technology |
|---|---|
| Backend API | Python · FastAPI · Uvicorn · Pydantic |
| Frontend | React 18 · Vite 5 |
| Dependency graph | d3-hierarchy (custom SVG) |
| SBOM output | CycloneDX 1.5 (cyclonedx-python-lib) · SPDX 2.3/3.0 (hand-built) |
| Vulnerability data | Bundled offline dataset + live OSV API · SQLite cache |
| PDF reports | reportlab |
| Binary parsing | pefile (PE) + hand-rolled ELF/wheel/JAR parsing |
| Auth | PyJWT + bcrypt |
| Persistence | SQLite (stdlib) |
| Desktop packaging | pywebview · PyInstaller · Inno Setup |
| CI/CD | `sbomgen` CLI with `--fail-on` gating |
