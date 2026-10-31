---
title: "Python acq v1: inspectable runtime and distribution feasibility"
description: "Bounded primary-source assessment of managed Python runtimes, offline dependencies, and existing installer channels"
status: draft
tier: 2
last_updated: 2026-10-31
---

# Python acq v1: bounded feasibility

## Verdict and scope

**Yes, conditionally:** acq can ship readable Python source and run with an
approved, prebuilt managed interpreter without users managing Python or compiling
anything at installation. This does **not** require freezing acq into an opaque
executable. It does require an explicit runtime supplier, deployment contract,
supported platform matrix, and security-update owner. [W][M][U][S]

**Not established:** a single Python.org portable-binary path covering native Windows, macOS, and Linux.
Windows has an official embeddable distribution; macOS has a system framework installer; Linux guidance
describes distro packages or source builds, not a generic portable Python.org binary. These are not
interchangeable artifacts. [W][M][U]

Scope is Python v1 replacing Bash for existing MSB and SBX users, with ordinary use after IT provisioning.
No language comparison or final packaging decision is made. Future API adapters are compatible in principle,
but neither their design nor implementation is assessed here.

Evidence labels distinguish **fact**, **repository observation**, **hypothesis/option**, and **human decision**. Feasibility is not fleet approval.

## Official CPython distribution, by operating system

| Platform | Fact: first-party path | Consequence for this scope |
|---|---|---|
| Windows | Python install manager from Python.org/Store; minimal embedded ZIP; NuGet packages also documented. Embedded ZIP carries executable, DLLs and bytecode standard library, but no pip/Tk/docs. [W] | Private runtime plus readable acq source is directly supported. Dependencies belong beside the embedded interpreter and must be supplied by the application distributor, not fetched at launch. |
| macOS | Signed/notarized universal2 `.pkg`, supporting Intel and Apple Silicon within its release-specific OS floor. Default installation requires administrator credentials and uses fixed `/Library/Frameworks`, `/Applications`, `/usr/local/bin` locations. [M] | Suitable for IT-provisioned runtime plus source. Not a portable, arbitrary-user-prefix archive; repackaging/relocation would need separate engineering and validation. |
| Linux | Most distributions provide Python; CPython documents distro packages and compiling source when the desired version is absent. [U] | IT can provision a suitable distro package or centrally built package. No-install-compilation is feasible only if the approved binary already exists; “Python is usually installed” is not a version/feature guarantee. |

**Windows subtleties:** the current install manager separates manager updates from runtime updates.
It supports predownloaded offline indexes and direct ZIP extraction without registration, launching
by explicit executable path. Its MSI installs the manager per machine but does not bundle runtimes;
normal managed runtimes remain per user. IT can extract a shared runtime with `--target` and supply
its registration/launcher. Default launch behavior can automatically install a missing runtime;
ordinary acq must not depend on that. [W: Offline installs, Advanced installation, Installing runtimes,
Configuration]

**macOS subtleties:** Apple-controlled `/usr/bin/python3` is documented as an often older/incomplete
developer-tools Python and must not be modified. Python.org's certificate-completion command downloads
certifi certificates: a provisioning/network step, not an acceptable surprise at ordinary runtime.
Enterprise trust/CA integration must be validated for the chosen runtime, not inferred from the language.
[M: Installation steps]

## Managed versus bundled options (not choices)

| Option/hypothesis | Ordinary-user experience | Ownership and unresolved cost |
|---|---|---|
| A. IT-managed interpreter + acq source/dependencies | Installer binds an approved explicit interpreter path; user types `acq`, not `python`/`pip` or activation commands. | IT patches base interpreter; acq maintains dependencies and compatibility. Must handle fleet versions and base-runtime replacement without breaking application environment. |
| B. App-private prebuilt interpreter + source/dependencies | OS-specific installer/archive places separate `runtime/`, `src/`, dependency and launcher trees. No global Python needed. | Distributor must patch interpreter and its native libraries, approve supplier, retain notices and publish provenance. Windows has a first-party embedded precedent; macOS/Linux need an additional approved build path. |
| C. Channel-specific hybrid | Homebrew-managed Python for brew users; private runtime or IT-managed path for other channels, all with identical acq source contract. | Preserves channels but multiplies environments; support matrix and patch responsibility must be explicit. |

For B, **supplier fact:** Astral's Python Build Standalone publishes portable Windows/macOS/Linux
distributions, usable directly from release archives without adopting uv. Install-only gzip archives
contain the runtime tree; full archives also include build artifacts and `PYTHON.json` metadata.
This documents **Astral's builds**, not CPython release-team binaries or existing approval here. [S][SD]

**Compatibility fact:** standalone GNU/Linux builds depend on glibc with a documented minimum;
musl builds target a different libc. Fully static musl cannot load dynamic third-party extensions.
CPU variants also differ; “portable” does not mean every Linux/CPU/OS version. Full metadata records
target, ABI, native dependencies and licensing; install-only archives omit `PYTHON.json`, so a
distributor should preserve equivalent release metadata. [SR][SD]

**Build option:** organization-built CPython packages move compilation into an approved build/release
environment, satisfying *no compilation at user installation*. They create a new binary-maintenance
obligation, not a zero-cost fallback. No such build has been attempted here. [U; inference]

## Repository observations and constraints

- `install.sh` selects existing Homebrew, then npm, then managed clone; default
  clone storage is `${XDG_DATA_HOME:-$HOME/.local/share}/acq`, with a launcher in
  `$HOME/.local/bin`. Consent gates PATH edits. Release-asset SHA and requested
  clone SHA have distinct meanings. [R1][R2]
- `package.json` exposes the Bash `acq` bin, limits OS to darwin/linux, and runs
  an installation-time wheelhouse download. It is not currently a native Windows
  Python distribution. `acq.ps1` delegates to Git Bash. [R3][R4]
- Existing provider-validator packaging pins PyYAML 6.0.3 and jsonschema 4.26.0
  plus transitive dependencies, with per-platform hash locks. Release workflow
  downloads binary wheels for Linux x86_64/aarch64 (Python 3.12), macOS
  arm64/x86_64 (3.12/3.13/3.14), Windows x64 (3.12). [R5][R6]
- The Windows release bundles Python.org **3.12.10 embedded amd64 ZIP**, verifies
  its pinned SHA-256, stages packages into `Lib/site-packages`, edits `._pth`,
  and retains `LICENSE.txt`. The installer verifies the nested archive and places
  it in `%LOCALAPPDATA%\acq\provider-input-validator`, with replacement/rollback.
  This proves an existing packaging design, **not a new live validation**. [R5][R7]
- macOS/Linux setup still finds an existing interpreter and creates a venv.
  `common.sh` can invoke that offline setup when the validator runtime is absent;
  no package download at create is not the same as no runtime setup at create.
  A venv depends on its base interpreter and is not generally relocatable. [R6][R8][V]
- The upstream validator-install change was merged on 2026-10-24, merge SHA
  `5192fa1901215c9fd273219bb10d95861afdc492`. Its stated result was normal-install
  validator usability; the local code shows Windows interpreter bundling, not
  interpreter bundling for every OS. The PR is context, not proof of fleet success.
  [R9]

**ADR implications:** ADR-0026 preserves inspect-first, no mandatory Homebrew,
user-space/non-elevating installation, consent and channel lifecycle. Its
“minimal new surface” assumption would need reconsideration for a managed Python
runtime; native Windows must replace the documented Git Bash preview arrangement.
ADR-0032 distinguishes installed pin from accepted version policy and checksum
integrity from provenance; retain those distinctions for Python. Its rejection of
vendoring concerns **MSB binaries**, not a universal prohibition on Python
bundling, but Python redistribution still needs an explicit new decision. [R1][R10]

## Dependencies, offline operation, pinning and maintenance

**Facts:** a wheelhouse allows offline dependency installation with compilation already performed.
Wheels are not necessarily portable: compatibility combines Python implementation/version, ABI,
OS/architecture and (Linux) libc baseline. `abi3` does not erase OS/native-library constraints;
free-threaded builds use different compatibility requirements. [P][T]

**Availability observation:** PyPI's version-specific file APIs list normal
CPython 3.12/3.13/3.14 wheels for both PyYAML 6.0.3 and rpds-py 2026.9.1 on
Windows x64/ARM64, macOS Intel/ARM64, and Linux glibc x64/ARM64. This is a bounded
metadata check of two native dependencies, not a complete v1 dependency audit,
hash approval, successful import or proof that existing locks include every wheel.
The repository's Windows lock/build remains x64/3.12 only. [F1][F2][R5]

**Proposed release contract:** exact interpreter version **and build identity**,
target OS/CPU/libc floor, dependency versions/artifact hashes, source revision,
launcher/import-path rules, third-party notices and provenance manifest. Explicit
IT install/update consumes complete staged artifacts; ordinary launch uses them
or fails with remediation, never pip resolution or automatic runtime downloads.
Build with binary-only dependencies, or produce missing wheels in the approved
build environment; missing wheels must not trigger local compilation. [P][T;
proposal supported by R5]

**Licensing fact:** PSF License v2 permits source/binary redistribution with retained license/copyright;
derivatives require a brief summary of Python changes. Incorporated software has additional licenses.
Do not relabel bundled Python or native wheels as acq's CC0 license. Standalone builds document
their own dependency license differences and provide notices/metadata. [L][SR][R3]

**Maintenance fact:** CPython's bugfix phase publishes binaries; security-only phase publishes
source-only updates. A long-lived exact binary pin is not a security-maintenance plan. The existing
3.12.10 embedded pin must be reassessed against current source/security updates by the new runtime
owner, without assuming the last Python.org binary receives later source fixes. [D][R5]

**Human operational decisions:** who monitors Python/native-library/wheel advisories, approves patches,
rebuilds/re-signs offline bundles, inventories fleet versions, retains rollback artifacts and retires
unsupported runtimes? IT-managed distro builds may carry vendor backports, so an upstream version
number alone is insufficient evidence of patch status. Backport verification belongs to that
distribution's security records; none were checked here.

## What distribution channels solve (and do not)

- **macOS `.pkg`:** first-party CPython demonstrates a signed/notarized package
  containing a prebuilt runtime. An acq package could carry source and dependencies
  or bind an IT-provisioned runtime. Package format does not pick a runtime,
  guarantee no-admin installation, provide ongoing security updates, or prove
  enterprise approval. Python.org's current package expressly needs admin and
  cannot use Installer `-domain` for arbitrary locations. acq's own signing,
  notarization and deployment arrangements remain unsettled. [M][R1]
- **WinGet:** discovers/downloads/installs/upgrades packages; supports ZIP,
  portable, MSI/MSIX/EXE formats, explicit version/architecture/scope selection.
  It invokes the selected installer: `--scope user` is not permission elevation
  avoidance if that installer cannot support user scope. App Installer may be
  missing, or registration incomplete after first login. Never assume availability
  on managed devices or treat it as permission to enable hypervisor features.
  Offline predownload is supported, but Store licensing/auth conditions and local
  manifest administrative enablement are distinct constraints. [G][GI][GD]
- **Homebrew:** can preserve runtime ownership with a declared Python dependency
  and application-local modules. Standard Python resource helpers can build source;
  bottles avoid build only on matching supported systems/prefixes. Preserve this
  channel where approved, but fail/give IT guidance rather than silently compile
  on a missing bottle. A minor-version formula is not an exact patch/build lock.
  [B][BB; final sentence inference]
- **npm and clone:** can remain delivery conveniences for existing users, not
  become Python-management chores. A clone needs an approved external runtime
  manifest/binding; an npm package needs pre-staged platform assets and launcher
  changes. Neither git checkout nor npm alone supplies CPython. Existing postinstall
  fetches mean today's npm package is not an air-gapped install bundle. [R2][R3]
- **Direct release archive/installer:** plausible fallback when package managers
  are unavailable; must contain/mirror all runtime/dependency assets for offline
  deployment. Preserve current consent/provenance behavior, while separating IT
  provisioning from ordinary runtime. [W][P][R1; proposal]

## Meaningful inspectability and decision gate

Proposed constraints: ship the **executed acq `.py` files**, not merely a link to
source or bytecode-only bundle; expose launcher-to-source and runtime paths, exact
revision/build metadata and checksums; keep dependencies and native components
identifiable. Include licenses and upstream source/build references (offline source
copies if IT requires them). Windows embedded stdlib is bytecode-only, so readable
acq does **not** imply every runtime component is readable in place. [W]
Signed binaries plus readable source do not prove reproducible builds. Editing
installed source may invalidate integrity/approval; supply a separate development
workflow rather than equating inspectability with production mutability.

Humans must settle: A/B/C ownership; approved suppliers; Python minor and patch policy;
supported Windows/macOS/Linux/CPU/libc matrix (including Windows ARM64 and musl scope);
writable versus IT-protected code location; CA/crypto requirements; offline mirror/signature
policy; and channel commitments. Native Python availability does not establish backend
virtualization, path, credential-store or MSB/SBX compatibility; those remain separate gates.

## Access context and verification limitations

AI-assisted, read-only research plus this single artifact; accessed 2026-10-31. Local snapshot:
`fe686ea7871bfc56f0290d5bebd4adb9fd01b17b`, package 4.1.1.
Owner-maintained documentation source was read on CPython `3.14` and other
projects' default/docs branches, then heads resolved through GitHub API; pinned
links below identify that access context, not selected runtime releases.
Documentation transport failed for Python docs, Microsoft Learn, Astral/pip docs
and Apple pages; GitHub-maintained source succeeded. Guessed source paths returning
404 were corrected by owner-repository directory discovery; Apple material was
not used as evidence. PyPI file metadata succeeded without downloading wheels.

Verification: source/claim cross-check and single-artifact content inspection; no installs, builds,
dependency scans, provisioning tests, service launches or Windows/macOS execution. Markdown lint
was unavailable locally; no dependency installation was attempted. Live portability, fleet approval,
exact runtime provenance/signature validation and end-to-end operation are **not verified**.

## Primary sources

[W]: https://github.com/python/cpython/blob/509eb96daeeaf89f58f4a92c2a78cabfffcbb16a/Doc/using/windows.rst
[M]: https://github.com/python/cpython/blob/509eb96daeeaf89f58f4a92c2a78cabfffcbb16a/Doc/using/mac.rst
[U]: https://github.com/python/cpython/blob/509eb96daeeaf89f58f4a92c2a78cabfffcbb16a/Doc/using/unix.rst
[V]: https://github.com/python/cpython/blob/509eb96daeeaf89f58f4a92c2a78cabfffcbb16a/Doc/library/venv.rst
[L]: https://github.com/python/cpython/blob/509eb96daeeaf89f58f4a92c2a78cabfffcbb16a/LICENSE
[D]: https://github.com/python/devguide/blob/a4da55860a469b4e4453f7c3cb2553ce4e38d03f/versions.rst#status-key
[S]: https://github.com/astral-sh/python-build-standalone/blob/4abc4b21cd5ffb145ba32b6d471e0b09eb6ad30b/README.md
[SD]: https://github.com/astral-sh/python-build-standalone/blob/4abc4b21cd5ffb145ba32b6d471e0b09eb6ad30b/docs/distributions.rst
[SR]: https://github.com/astral-sh/python-build-standalone/blob/4abc4b21cd5ffb145ba32b6d471e0b09eb6ad30b/docs/running.rst
[P]: https://github.com/pypa/pip/blob/ec43c48b2f2de7bfaac32bee85ee33a9e496db07/docs/html/topics/repeatable-installs.md
[T]: https://github.com/pypa/packaging.python.org/blob/6141e574ede0199fec669fa98beaad8c2f9eb676/source/specifications/platform-compatibility-tags.rst
[F1]: https://pypi.org/pypi/PyYAML/6.0.3/json
[F2]: https://pypi.org/pypi/rpds-py/2026.9.1/json
[G]: https://github.com/MicrosoftDocs/windows-dev-docs/blob/a3812316d9d6678be4e526ffb4b1c4f3958d956d/hub/package-manager/winget/index.md
[GI]: https://github.com/MicrosoftDocs/windows-dev-docs/blob/a3812316d9d6678be4e526ffb4b1c4f3958d956d/hub/package-manager/winget/install.md
[GD]: https://github.com/MicrosoftDocs/windows-dev-docs/blob/a3812316d9d6678be4e526ffb4b1c4f3958d956d/hub/package-manager/winget/download.md
[B]: https://github.com/Homebrew/brew/blob/1e4523e5643933fb92ee2e336c58dfab13e34c65/docs/Language-Specific-Formulae.md
[BB]: https://github.com/Homebrew/brew/blob/1e4523e5643933fb92ee2e336c58dfab13e34c65/docs/Bottles.md
[R1]: https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/docs/adr/0026-installation-and-distribution.md
[R2]: https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/install.sh
[R3]: https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/package.json
[R4]: https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/acq.ps1
[R5]: https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/.github/workflows/release.yml
[R6]: https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/scripts/setup-provider-input-validator
[R7]: https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/install.ps1
[R8]: https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/acq.backends/common.sh
[R9]: https://api.github.com/repos/GSA-TTS/agentic-coding-quickstart/pulls/619
[R10]: https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/docs/adr/0032-msb-version-policy-and-migration-recovery.md
