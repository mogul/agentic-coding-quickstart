---
title: "Windows ordinary-user operating envelope"
description: "Evidence, provisioning boundaries, and live qualification gates for native Windows MSB and SBX"
status: draft
tier: 2
last_updated: "2026-10-31"
---

# Windows ordinary-user operating envelope

## Conclusion and scope

After IT enables the required hypervisor and approves the execution/network policy,
**both backends have native Windows, user-scoped installation paths**. This does
not prove that every restricted device permits them, or that the current `acq`
agent flow works. MSB Windows remains preview/beta; Docker independently documents
SBX on Windows 11 x64 and explicitly permits no-admin per-user installation [M1, D1].
Neither requires Docker Desktop, WSL, a Python SDK, or a full Hyper-V management
installation merely to use its native CLI [M1, M3, D1].

**Complete agent readiness: NOT QUALIFIED.** This research ran on Linux, not a
Windows host. No installers, backends, privileged operations, or live agent flows
were executed. Local reads and authorized public HTTPS reads are the evidence.
Research is AI-assisted; human review and Windows validation remain necessary.

Access context: 2026-10-31. Local source HEAD is
`fe686ea7871bfc56f0290d5bebd4adb9fd01b17b`, installer default `acq` 4.1.1;
unrelated existing working-tree changes were left untouched. Local MSB pin is
0.7.7 (published 2026-10-05), minimum 0.6.9, excluding 0.7.0–0.7.2; fixed line
starts at 0.7.3. SBX adapter floor is 0.39.0, **not a Windows-support floor**;
latest SBX release observed is 0.50.1 (2026-10-28), with Windows AMD64 MSIs [L1, M2, D2].
Use exact approved releases/checksums; accepting newer versions is not qualification.

## Five separate readiness gates

| Gate | What establishes it | What does not establish it |
| --- | --- | --- |
| Installation | Approved artifacts copied/installed in writable paths; correct binaries resolve | Installer exit 0 alone |
| Host runtime | WHP API usable and a real guest boots/executes under a standard token | Feature checkbox, version output, WSL2 installed |
| Filesystem/process portability | Real mounts, path guards, locks, signals, teardown tested | Linux stubs, installing native Python |
| Credential transport | Host protected store, destination binding, TLS/proxy/VPN, authenticated guest request | Host browser access, DPAPI round-trip, guest boot |
| Complete agent | Pinned kits validate/install; agent answers and edits/tests a mounted project; restart/cleanup works | Any single preceding gate |

## Support and privilege matrix

“Conditional user” means technically user-scoped **and** dependent on IT policy;
it does not authorize bypassing policy. Unknowns are deliberately explicit.

| Operation | MSB native Windows | SBX native Windows |
| --- | --- | --- |
| OS/CPU | Upstream 0.7.7: Windows 10+, x64/ARM64 preview; `acq` package/runtime reviewed here is x64 | Official: Windows 11, 64-bit Intel/AMD; ARM64 support not documented [M1, D1] |
| Firmware virtualization / VDI nesting | IT/platform provisioning; cannot fix disabled firmware from user-space | IT/platform provisioning; nested virtualization required in a VM/VDI [D1] |
| Enable WHP, required reboot | Administrator: `HypervisorPlatform`; upstream `doctor --fix` opens elevated PowerShell | Administrator: official elevated optional-feature command [M1, M3, D1] |
| Full Hyper-V / WSL2 / Docker Desktop | Not required by documented native route; WHP differs from `VirtualMachinePlatform` | Docker Desktop/Engine not required; WHP required for local sandboxes [D1] |
| Install backend | Conditional user: ZIP binary + DLL under `%USERPROFILE%\.microsandbox` or writable `MSB_HOME` | Explicit no-admin per-user `DockerSandboxes.msi`, `%LOCALAPPDATA%\DockerSandboxes`; WinGet route documented [M4, D1] |
| System install/update | Any protected prefix needs IT; user installer does not grant write permission | Machine MSI under `%ProgramFiles%\DockerSandboxes` and system PATH needs elevated deployment [D1] |
| Host service/driver setup | Inspected ZIP installer installs no custom driver/service; WHP is OS infrastructure; runtime child processes rather than setup daemon [M4, M5] | Per-user MSI supports no-admin install; daemon exists. Exact service registration/token/driver inventory is **unknown**, proprietary backend not audited [D1, D3, D4] |
| Create/run/exec/stop | Conditional user after WHP; reported guest boot exists, current full wrapper blocked | Documented local workflow after install/sign-in; restricted-host ordinary-token runtime **not measured** [P1, D1, D3] |
| Update backend | Conditional user: writable root, downloads and file unlocks; `self update` uses Limited scheduled task + PowerShell/COM [M5] | Per-user upgrade via approved package channel expected within scope; exact unattended upgrade/daemon-lock behavior **not qualified**; machine upgrade IT-owned [D1] |
| Publish ports | User-mode listeners; firewall policy may prevent use; post-hoc `acq` route also needs SSH/keygen and reliable teardown | Port publication supported by adapter; Windows listener/firewall behavior **not qualified** [M1, L1] |
| Store credentials | `acq` DPAPI CurrentUser via Windows PowerShell; no service/admin needed under permissive policy | `acq` DPAPI store plus SBX native Windows Credential Manager/proxy; distinct stores [L2, D4, W4] |
| Install/update guest tools | Inside guest, according to guest UID/package/network policy; not host elevation | Same distinction; private Docker daemon is guest infrastructure [D3] |

## One-time IT provisioning is more than a feature checkbox

- [ ] Approve Windows edition/build/architecture and virtualization on physical
  hardware; provide nested virtualization if VDI is permitted. Start with Windows
  11 x64 for the intersection of documented backend and `acq` packaging support.
- [ ] Enable WHP through the approved management mechanism, reboot, and verify
  usable WHP under a **non-elevated** account. Do not grant Hyper-V Administrators
  membership or install full Hyper-V speculatively. WHP exposes user-mode APIs;
  local installer probes `WHvCreatePartition`, upstream MSB 0.7.7 doctor checks
  `WHvGetCapability`/HypervisorPresent. Neither alone proves full VM setup [W1, M3].
- [ ] Approve exact executable/DLL/script inventory: backend CLI and runtime,
  Git Bash/MSYS DLL and utilities, Windows PowerShell, `acq.cmd`/`.ps1`/Bash,
  embedded validator Python and packages, `git`, `curl`, native `jq`, and optional
  SSH/keygen. Native binary execution and interpreter permission are separate.
- [ ] Evaluate AppLocker/WDAC/App Control, software restriction policy,
  SmartScreen/download provenance, antivirus/EDR, controlled-folder access,
  Windows Installer/WinGet restrictions and user PATH policy. Directory writability
  does not imply execution or DLL-load permission. Actual policy is **unknown**.
- [ ] Approve PowerShell script execution and required .NET calls. AppLocker/WDAC
  can induce ConstrainedLanguage, restricting arbitrary `Add-Type`, .NET and COM.
  Group Policy execution settings outrank user/process settings. Allow approved
  scripts/components through IT; do not use `Bypass`, alternate interpreters or
  shell entry points as policy workarounds [W2, W3].
- [ ] Decide if MSB's user-level scheduled-task registration, logon/retry triggers,
  PowerShell encoded commands, COM task ACL adjustment and helper execution from
  its state root are allowed. `RunLevel Limited` is not a guarantee of policy
  permission. If prohibited, IT can own approved stopped-runtime bundle updates
  instead; test this path rather than promising self-update [M5, L1].
- [ ] Provide local writable code/cache/state locations, adequate disk/RAM/quota,
  NTFS ACLs and exclusions only where approved. Prefer short local nonsynced paths;
  do not presume SMB, redirected profiles or OneDrive behave as local storage.
- [ ] Approve artifact/registry/auth/API destinations and enterprise proxy/TLS/VPN
  requirements. Downloads may redirect beyond GitHub's visible URL. Include pinned
  kits/playbook and agent/package registries, not only the inference endpoint.
  Docker additionally requires sign-in and lists Docker auth/registry/diagnostic/
  telemetry domains; IT decides data/account/governance acceptability [D5].
- [ ] Preconfigure firewall/listener policy if ports are needed; keep development
  binds on loopback. Do not ask users to approve broad public-network access [M1].
- [ ] Assign backend/OS/support/update owners and an approved recovery process.
  Feature/driver/policy changes remain IT work after initial provisioning.

## User-writable installation and interpreter envelope

Local `install.ps1` writes `acq` under `%LOCALAPPDATA%\Programs\acq`, backend
under `%USERPROFILE%\.microsandbox\{bin,lib}`, and validator under
`%LOCALAPPDATA%\acq\provider-input-validator\provider-input-validator.current`.
It checksum-verifies releases, changes **user** PATH with consent, and does not
enable WHP/elevate/reboot. Missing Git triggers `winget install Git.Git` without
an explicit scope/version; alternate MSB WinGet ID is likewise not a pinned
no-admin promise. Preprovision approved Git or use its reviewed user install [L1, G1].
Protected/machine installs cannot be upgraded by a user simply rerunning this script.
Important runtime distinction: `acq_backend_prepare` automatically attempts
`msb doctor --fix` after a failed doctor check; upstream Windows fix uses UAC
elevation. Therefore the **installer's** no-elevation promise does not establish
a no-prompt runtime failure path. IT/user qualification must verify this case;
`ACQ_SKIP_MSB_DOCTOR` skips checks, not the underlying WHP requirement [L3, M5].

The launcher is `acq.cmd` → in-box `powershell.exe -File acq.ps1` → Git for
Windows Bash → native `msb.exe`. It rejects Windows' WSL `bash.exe` shim. PowerShell
7 alone is not the configured credential-store runtime; Git Bash is not WSL [L1, L2].
Source-checkout Bash files need LF. Record repository-specific `core.autocrlf`
and `.gitattributes`; do not prescribe global Git changes as a generic fix.

Official Windows ZIP packaging embeds CPython **3.12.10** and pinned PyYAML 6.0.3 /
jsonschema 4.26.0. The source checkout lacks release-built embedded assets by
default; its validator setup accepts Windows Python **3.12** with venv/pip and
hash-locked packages, or an explicit interpreter with the required packages [L3].
Source setup's symlink-based activation must also be tested under actual MSYS
link behavior; the release installer instead uses staged directory moves.
**Installing Python does not enable WHP, authorize binaries, add POSIX APIs,
repair MSYS process semantics, or make the backend ready.**

## Paths, processes and credentials: current blockers

- ADR-0029 requires native host paths such as `C:/work/project` for mount sources
  and POSIX guest paths such as `/c/work/project` for targets, cwd and guest env.
  `_acq_msb_cli` scopes `MSYS2_ARG_CONV_EXCL='*'` to MSB. Do not export it globally;
  Git/SSH may rely on conversion. Launcher UNC forwarding is not a verified UNC
  guest mount. SBX Windows mount translation must be validated independently [L4].
- NTFS is not POSIX: `chmod 0600`, executable bits, symlinks/junctions, casing,
  rename/open-handle sharing and signal/PID behavior require separate contracts.
  Windows long paths require application opt-in plus managed OS configuration;
  neither adding Python nor enabling one registry value fixes all tools [W5].
- Local `common.sh` still uses `O_DIRECTORY`, `O_NOFOLLOW`, `dir_fd` in safe
  directory/snapshot helpers and `ps -o` for guardian/process identity [L3].
  Native Windows Python/MSYS cannot supply those POSIX semantics. The pinned
  Patterns validator also requires `st_mode & 0o111` on its normalizer [K1].
  These are shared wrapper blockers, not proofs of either backend failing WHP.
- Local MSB provider-input read-only mount trust explicitly rejects Windows as
  an unreviewed filesystem backend (`_acq_msb_ro_mount_platform_ok`). Preserve
  validation/fail-closed guarantees; do not substitute permissive copying [L3].
- ADR-0028 protects `acq` secrets using DPAPI CurrentUser, separate from backend
  transport. Reported Windows round-trip exists; cross-user negative test was
  not performed there. Microsoft documents same-logon/usually-same-machine scope,
  with roaming-profile exceptions. It is not isolation from all same-user code [L2, W4].
- Missing/blocked PowerShell must not be treated as permission to use plaintext
  file fallback on Windows. Check protected backend before entering real secrets.
  Never put values in argv, screenshots, history, env dumps, fixtures or reports.
- MSB binds host credentials to destinations for swap-on-wire; SBX uses its
  host proxy/credential store. VPN reachability, TLS interception CA trust, registry
  auth and service binding must work across both host and guest requests. A host
  authentication check is not a guest agent transaction. Revalidate rotation/
  existing-sandbox behavior for the selected versions, not older assumptions [D4].

## Recent upstream observations — reported, not reproduced here

| Evidence | Reported scope/outcome | Limit |
| --- | --- | --- |
| Preview installer [P1], merged 2026-09-21 | Windows 11 Home build 26200; MSB 0.6.18; WSL2/VMP present; non-elevated WHP probe, ZIP/PATH and native guest boot passed | Full first-run stopped at USAi key/network gate; not managed-device or 0.7.7 qualification |
| CRLF ports [P2], merged 2026-10-29 | Windows validation comment: native jq 1.8.2, Git 2.55.0, Bash 5.3.15, Python 3.12.10; actual CRLF bug fixed | Backend stubs used; file 17/18 passed, unrelated teardown test failed on both revisions |
| Provider-input guard [P3], open at access | Windows-aware handle/reparse-point checks and MSYS process fallbacks; reports ports file 18/18 | Snapshot POSIX API gap remains; projection file has failures; create/TERM test hung >300s; short-name check not live-qualified |
| Windows/MSYS port [P4], open at access | Reports real provisioning iterations through credential selection, staging and kit fetch; cache-path/anonymous-Git fixes proposed | Pinned normalizer exec-bit guard remains; overlaps guard work; initial permissive path fallback is not an approved safety contract |

PR text is evidence of a specific reported experiment, not acceptance criteria met.
Do not combine open heads with current release behavior or promote a partial run
to full readiness. Existing ADR decisions remain applicable; a Python rewrite or
alternative backend trust model needs a separately approved decision.

## Required live Windows qualification (future, separately authorized)

1. **Capture envelope:** UTC timestamps, exact source/artifact hashes, Windows
   edition/build/CPU, physical/VDI status, non-admin token evidence without personal
   identifiers, IT policy baseline, shell/binary paths and versions. Record execution
   policy/language mode and interpreter platform; no environment dumps.
2. **Install:** fresh standard account, reviewed pinned artifacts, checksum failure
   negative test, writable-path/default/custom/spaced-path installs, PATH decline/
   acceptance and new terminal. Missing prerequisite must stop without elevation.
   Repeat independently for MSB and SBX per-user MSI; capture any UAC/security block.
3. **Host runtime:** normal `msb doctor` (not `--fix`), native create/exec/stop/start
   smoke on each backend with a recorded image digest. SBX sign-in must be approved.
   Capture output/exit codes and host process identities; guest boot must be real.
4. **Filesystem/process:** mounted bidirectional edit, cwd, spaces/Unicode/long paths,
   copy/clone, CRLF native jq, links/junction rejection, read-only enforcement,
   locks/concurrency, PID reuse identity, Ctrl-C/TERM, failed create and orphan-free
   stop/remove. Test guard changes and fetched normalizer contract on the exact pins.
5. **Credentials/network:** human enters scoped credentials through approved prompts;
   verify protected store and second-account denial without displaying plaintext.
   On approved network/VPN, observe authenticated host and guest API success,
   denied destination, invalid/missing key, rotation and existing-sandbox behavior.
   Capture only sanitized statuses; never request token/key values in research output.
6. **Complete onboarding:** pinned kits/validator pass, full `acq run` reaches the
   chosen agent, receives a real response, edits a disposable mounted project and
   runs its test, then reattach/restart/cleanup and (if needed) loopback port lifecycle.
7. **Update/recovery:** approve upgrade separately; stop-own-process bundle replacement
   and MSB task-based self-update completion/log/task cleanup, SBX per-user upgrade,
   new terminal resolution and existing state compatibility. Test policy denial.
   Record PASS/FAIL/BLOCKED per gate; do not call a skip a pass.

Human decisions remaining: target managed-device cohort; approved backend/version
and distribution; user vs IT update ownership; PowerShell/task/interpreter policy;
local paths/network/account approvals; adoption/sequence of open portability changes;
required Windows security invariants and sign-off. Until this evidence exists,
publish a **conditional operating envelope**, not “ordinary users fully onboarded.”

## Primary-source index

- **L1:** Local `install.ps1` (version/path/WHP/install functions), `acq.cmd`, `acq.ps1`, `acq.backends/sbx.sh:64`; [source snapshot](https://github.com/GSA-TTS/agentic-coding-quickstart/tree/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b).
- **L2:** Local `acq.backends/secret-store.sh:206–236`, [ADR-0028](https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/docs/adr/0028-windows-native-secret-storage.md).
- **L3:** Local `common.sh:4148–4211,4394–4419,4542,4645,4694+`, `msb.sh:845–881,1988–1995`, `scripts/setup-provider-input-validator`, `.github/workflows/release.yml:86+`.
- **L4:** [ADR-0029](https://github.com/GSA-TTS/agentic-coding-quickstart/blob/fe686ea7871bfc56f0290d5bebd4adb9fd01b17b/docs/adr/0029-windows-host-guest-path-forms.md); local `docs/agents/domain.md` read; no glossary present.
- **M1:** [MSB Windows docs, v0.7.7](https://github.com/superradcompany/microsandbox/blob/v0.7.7/docs/troubleshooting/windows.mdx); [README](https://github.com/superradcompany/microsandbox/blob/v0.7.7/README.md).
- **M2:** [MSB 0.7.7 release/assets](https://github.com/superradcompany/microsandbox/releases/tag/v0.7.7).
- **M3:** [MSB WHP prerequisite source](https://github.com/superradcompany/microsandbox/blob/v0.7.7/sdk/rust/lib/setup/windows.rs).
- **M4:** [MSB Windows installer source](https://github.com/superradcompany/microsandbox/blob/v0.7.7/scripts/install.ps1); upstream installer can offer elevated doctor fix, unlike the wrapper's installer.
- **M5:** [MSB self-management/task source](https://github.com/superradcompany/microsandbox/blob/v0.7.7/crates/cli/lib/commands/self_cmd.rs), especially `register_windows_self_swap_task` and `run_windows_powershell`.
- **D1:** [Official SBX install](https://docs.docker.com/ai/sandboxes/install/); [documentation snapshot](https://github.com/docker/docs/blob/53581df94e05ad4116a8d31e47ea098289fa5498/content/manuals/ai/sandboxes/install.md).
- **D2:** [SBX 0.50.1 release/assets](https://github.com/docker/sbx-releases/releases/tag/v0.50.1); [tagged README](https://github.com/docker/sbx-releases/blob/v0.50.1/README.md).
- **D3:** [Official SBX architecture](https://docs.docker.com/ai/sandboxes/architecture/), retrieved 2026-10-31; mutable docs, not a backend-source audit.
- **D4:** [Official credential docs snapshot](https://github.com/docker/docs/blob/53581df94e05ad4116a8d31e47ea098289fa5498/content/manuals/ai/sandboxes/configuration/credentials.md).
- **D5:** [Official FAQ/sign-in/domains snapshot](https://github.com/docker/docs/blob/53581df94e05ad4116a8d31e47ea098289fa5498/content/manuals/ai/sandboxes/faq.md).
- **W1:** [Microsoft WHP API documentation snapshot](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/85e96f5075e548ca24bc252e9f5ac0f39c318ef8/virtualization/api/hypervisor-platform/hypervisor-platform.md).
- **W2/W3:** Microsoft [language modes](https://github.com/MicrosoftDocs/PowerShell-Docs/blob/0061dc7f35abfa6bb5ffabaada1650163761f2e1/reference/5.1/Microsoft.PowerShell.Core/About/about_Language_Modes.md) and [execution policies](https://github.com/MicrosoftDocs/PowerShell-Docs/blob/0061dc7f35abfa6bb5ffabaada1650163761f2e1/reference/5.1/Microsoft.PowerShell.Core/About/about_Execution_Policies.md).
- **W4:** [Microsoft DPAPI API docs](https://github.com/MicrosoftDocs/sdk-api/blob/docs/sdk-api-src/content/dpapi/nf-dpapi-cryptprotectdata.md), retrieved 2026-10-31, mutable branch.
- **W5:** [Microsoft long-path documentation snapshot](https://github.com/MicrosoftDocs/win32/blob/ca8274decead49403d70ceeed4af8d4ca74eaf11/desktop-src/FileIO/maximum-file-path-limitation.md).
- **G1:** [Git for Windows installer snapshot](https://github.com/git-for-windows/build-extra/blob/2f63027cca82e0bd0d6aa9d79d52f5a0d7dea8cc/installer/install.iss), non-admin/HKCU branches; not a pinned deployed Git version.
- **K1:** [Pinned Patterns validator](https://github.com/GSA-TTS/agentic-coding-patterns/blob/7dd2ddc034b6597e97f450fa716e7d47c9a38287/scripts/validate_provider_input_contract.py).
- **P1–P4:** [Preview installation](https://github.com/GSA-TTS/agentic-coding-quickstart/pull/463); [CRLF live-Windows comment](https://github.com/GSA-TTS/agentic-coding-quickstart/pull/663#issuecomment-6489929880); [guard PR](https://github.com/GSA-TTS/agentic-coding-quickstart/pull/667) head `262a53a4f3618004f9d16acfd9fec431513a6acf`; [MSYS port PR](https://github.com/GSA-TTS/agentic-coding-quickstart/pull/644) head `2b6f62ccc4fc13aa8a415e5f1e426ce27c269b89` at access.

Retrieval limitations: Microsoft Learn HTTPS requests failed; Microsoft-owned GitHub
documentation supplied W1–W5 instead. Some moved Docker URLs returned 404; official
documentation source supplied D4/D5. No managed-device policy, Windows runtime,
binary/driver audit, complete agent flow, or live upgrade was validated here.
Artifact verification: frontmatter/line-budget/whitespace checks PASS;
`git diff --check` PASS (untracked artifact checked separately). Markdown lint
BLOCKED: no installed linter; no dependencies installed. Code tests/build/SAST/SCA
not applicable to this documentation-only research; Windows live gates BLOCKED.
