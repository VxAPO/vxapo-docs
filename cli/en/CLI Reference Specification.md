# CLI Reference Specification

> **Purpose**: Define the specification boundary, current capabilities, modification path, and execution constraints for the VxAPO CLI (`vxapo-cli`).
> **Positioning**: The CLI is a **developer tool** (see `overview/Project Overview.md` "three-layer separation"). It depends on `vxapo-driver` (as a library) and operates devices through the **install-layer API**; it never touches pipeline/RT. It targets debug automation and P0 end-to-end verification, not end users.
> **Basis**: Source read of `D:\APO_Project\VxAPO\vxapo-cli` (v0.3.0) + `D:\APO_Project\VxAPO\vxapo-driver` (2026-08-13).

---

## 1. Current CLI state (source read)

### 1.1 File structure (vxapo-cli)

```text
vxapo-cli/
├── Cargo.toml          # winreg = "0.52" + vxapo-driver (path = "../vxapo-driver")
│                       # + vxapo-protocol (path = "protocol") + serde/serde_json + windows
├── protocol/           # vxapo-protocol: the --json output contract shared with the App
└── src/
    ├── main.rs         # entry: no args -> interactive menu; subcommand -> automation
    ├── args.rs         # subcommand argument parsing (install/stale/config/snapshot/effects)
    ├── commands.rs     # command layer: list/status, install, uninstall, config, snapshot,
    │                   #   register, effects, help
    ├── commands/       # install.rs / uninstall.rs / status.rs / snapshot.rs / register.rs /
    │                   #   convert.rs / effects.rs
    ├── tui.rs          # interactive terminal menu (viewer mode + driver mode)
    ├── verify.rs       # install --verify pipeline (write -> full restart -> pipe -> scoring)
    ├── i18n.rs         # bilingual zh/en text (tr!)
    ├── display.rs      # endpoint list/detail printing
    ├── endpoint.rs     # standalone winreg enumeration (kept for viewer mode)
    ├── knowledge.rs    # known GUID/CLSID knowledge base
    ├── probe.rs        # probe helpers
    ├── reg.rs          # registry read helpers
    └── regdump.rs      # endpoint registry dump ([x] command)
```

### 1.2 Current commands

| Command | Behavior | Status |
|---------|----------|--------|
| no-argument launch | interactive menu: 1 viewer mode / 2 driver mode / q quit | implemented |
| `list` / `status` | enumerate devices and show GUID, version, mode, 5 slots, EAPO/lost-slot status; supports `--json` (includes `device_id`, the device instance ID) | implemented |
| `install -d <device> [--mode ...] [--no-child] [--verify] [--timeout=<sec>] [--progress-file=<path>]` | install/reinstall (auto mode detection, snapshot baseline, automatic driver COM registration, automatic config import; `--verify` runs the verification loop, see 5.1b) | implemented |
| `uninstall -d <device>` | uninstall (stop audio service, clear slots, read-back verification, snapshot diff; falls back to stale cleanup when the endpoint key is already gone) | implemented |
| `register` | locate the bundled `vxapo_driver.dll` and (re)register the CLSID→DLL binding (idempotent; also run automatically before every install) | implemented |
| `effects schema` | print the driver effect parameter specification (`effect_param_specs`) as JSON (`--json`) or a table; feeds the App's generated effect metadata | implemented |
| `config set/show/convert` | write/read `C:\ProgramData\VxAPO\{GUID}\config.toml`; `convert` migrates legacy EAPO-style txt to TOML | implemented |
| `snapshot diff/restore/create` | registry baseline snapshot, diff, restore (`changes` = `diff` alias, `baseline` = `create` alias) | implemented |
| `stale list/migrate/cleanup/fix-acl` | stale-GUID records: list, migrate to the current endpoint, clean up orphan records, repair config ACL | implemented |
| `[0-N]` select endpoint | viewer-mode endpoint detail submenu | implemented |
| `[x]` registry dump | dump endpoint registry keys | implemented |
| `[r]` refresh / `[q]` exit | re-enumerate / exit | implemented |

**Status**: the CLI has install/uninstall/config/snapshot/status capabilities and relies on
`vxapo-driver` APIs such as `install::device::info::enumerate_devices` and
`install::selector::operation`; it is no longer a pure interactive diagnostic tool.

> `stale` (and `register` / `effects`) are implemented but not yet listed in the
> `vxapo-cli help` text output (`print_help` is not synced); typing `stale list` and friends
> in the interactive menu works.

---

## 2. Reusable driver APIs (confirmed by source read)

`vxapo-driver`'s `lib.rs` exposes a small facade (all modules are `pub(crate)`); the CLI can
only call what the facade exports:

### 2.1 Install/uninstall (install/selector/operation.rs)

```rust
pub struct InstallConfig {
    pub install_premix: bool,
    pub install_postmix: bool,
    pub install_mode: InstallMode,       // SfxEfx default
    pub use_original_apo_premix: bool,   // keep the predecessor PreMix as child APO
    pub use_original_apo_postmix: bool,  // keep the predecessor PostMix as child APO
    pub allow_silent_buffer: bool,
    pub auto_adjust: bool,               // default false
}
// default_config(): SfxEfx, install both directions, use_original both false,
// allow_silent = true, auto_adjust = false

// Note 47: full 7-step registry write (transaction-protected, automatic rollback on failure)
// + v8.5 full/non-full decision
pub fn write_install_config(device_guid: &str, device_name: &str, connection_name: &str,
                            config: &InstallConfig) -> Result<()>;
// Pure registry write (no tail endpoint restart / AudioSrv ensure / CoCreateInstance self-check) —
// install --verify calls only this, then performs the whole-service restart + pipe verification itself.

// install_endpoint = write_install_config + tail (pnputil endpoint restart + ensure AudioSrv
// running) + optional CoCreateInstance self-check (verify=true; non-verify callers keep it)
pub fn install_endpoint(device_guid: &str, device_name: &str, connection_name: &str,
                        config: &InstallConfig, verify: bool) -> Result<()>;
pub fn uninstall_endpoint(device_guid: &str) -> Result<()>;
pub fn migrate_install(from, to, config_from, snapshot_from) -> Result<MigrationReport>;
```

> **v8.5 full-install decision built in**: `install_endpoint` starts by checking whether
> `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}` exists — absent = full backup path,
> present = non-full (a lost-slot reinstall overwrites the backed-up child APO with the latest
> predecessor).

### 2.2 Device query (install/device/info.rs + slots.rs)

```rust
pub struct DeviceInfo {
    pub endpoint: Option<EndpointInfo>,
    pub install_mode: InstallMode,
    pub slots: [SlotValue; 5],
    pub format: Option<AudioFormat>,
    pub installed_version: String,
}
// is_installed() / can_be_upgraded() / is_disabled() / is_unplugged()

pub fn enumerate_devices() -> Result<Vec<DeviceInfo>>;   // the single device-enumeration entry
pub fn find_endpoint_path(device_guid: &str) -> ...;     // endpoint key locator (uninstall stale fallback)
pub fn detect_mode_for_guid(device_guid: &str) -> InstallMode;  // EAPO three-tier auto-detection

// Slot + install info area (install/device/slots.rs)
pub enum ChildApoKind { PreMix, PostMix }
pub fn child_apo_key_exists(device_guid: &str) -> bool;                    // full/non-full decision input
pub fn read_child_apo_guid(device_guid: &str, kind: ChildApoKind) -> Option<GUID>;
```

> `read_all_slots` / `get_original_pre_mix` / `get_original_post_mix` exist inside the driver
> but are **not exported through the facade**; the CLI cannot call them directly — slot values
> come back inside `DeviceInfo.slots`.

### 2.3 audiodg (install/audiodg.rs)

```rust
pub fn ensure_can_load() -> Result<()>;   // DisableProtectedAudioDG check
pub fn stop_audio_service_with_dependents(stop_timeout_secs: u32) -> Result<()>;
// Stop AudioSrv's active dependent services (e.g. AudioEndpointBuilder) first, then AudioSrv; poll STOPPED
pub fn start_audio_service_with_dependents(start_timeout_secs: u32) -> Result<()>;
// Start AudioSrv and poll RUNNING (one retry after 5 s without progress), then start dependents
pub fn restart_audio_service_wait(stop_timeout_secs: u32, start_timeout_secs: u32) -> Result<()>;
// Dependency-aware whole-service restart (reused by uninstall finalisation / normal install finalisation)
pub fn ensure_audio_service_running() -> Result<()>;   // idempotent
pub fn wait_for_audiodg_exit(timeout_ms: u32) -> bool; // event-driven audiodg exit wait
```

### 2.4 config layer (config/parser.rs)

The driver's `ConfigParser` (`parse_file` / `parse_string`, spec + filter chain) is **not
exported through the facade**. CLI-side read-back verification is therefore **file-level only**
(the file was written and can be read back); DSP semantic validation is performed by the driver
itself when it parses the config for hot reload.

---

## 3. CLI boundary (three-layer separation + P0-7 positioning)

| Layer | CLI can do | CLI does not do |
|-------|------------|-----------------|
| install-layer API | call `write_install_config` / `install_endpoint` / `uninstall_endpoint` / `enumerate_devices` / `audiodg::*_with_dependents` through vxapo-driver | write the registry by itself (driver transaction); SCM service operations go through the driver audiodg module |
| slot-loss detection | detect the **installation-mode slots** via `enumerate_devices` + slot values (v8.5) | prompt reinstall + trigger `install_endpoint` to overwrite the backed-up child APO; never writes childApoPath directly |
| config management | `config set` writes `C:\ProgramData\VxAPO\{GUID}\config.toml` (v8.9 system-level root, TOML since v9.11); `config show` read-back verification; `config convert` one-time legacy txt → TOML migration | parse DSP semantics (read-back consistency is file-level only) |
| pipeline/RT | **never touched** | no real-time audio processing |
| rollback | snapshot before driver changes (registry state) | — |

**Verification boundary (P0-7)**: `config show` verifies "the file was written and can be read
back"; actual DSP hot-reload effectiveness is verified manually by listening (P0-4); complete
child-delegation verification requires real audiodg + a registered APO (P0-6 legacy, covered by
CLI end-to-end testing).

## 4. CLI evolution path (phased)

### Phase A: dependency integration + diagnostics (prerequisite for P0-7)

1. **Add the vxapo-driver dependency to Cargo.toml** (path = `../vxapo-driver`) — the CLI switched
   from standalone winreg enumeration to unified enumeration via
   `install::device::info::enumerate_devices` (removing duplicated logic from the self-implemented endpoint.rs)
2. **status/list commands**: call `enumerate_devices` to show the device list + `is_installed` /
   `is_disabled` / `installed_version` / install mode / 5 slot values (**annotating which slots
   are taken over**)
3. **Slot-loss detection wired in (v8.5 product intent)**: on CLI startup / device switch, detect
   that `DeviceInfo.install_mode`'s premix/postmix slot values ≠ VxAPO CLSID → prompt
   "reinstall needed" → confirm → `install_endpoint` (internally `child_apo_key_exists` decides
   full/non-full → lost-slot reinstall overwrites the backed-up child APO with the latest predecessor)

### Phase B: core commands (P0-7 end to end)

#### 4.4.1 Command parameter definitions (added 2026-08-04)

**`<device>` parameter: accepts two forms, normalized by `resolve_device` into `(device_guid, device_name, connection_name)`**

| Form | Syntax | Description | Priority |
|------|--------|-------------|----------|
| **GUID (canonical, recommended)** | `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}` | stable unique; equals `EndpointInfo.endpoint_guid` (the registry `MMDevices\Audio\Render\{guid}` key name); passed directly as `install_endpoint`'s first argument | GUID format matched first (starts with `{`) |
| **enumeration index** | non-negative integer `0..N-1` | index into the `list`/`status` output order (interactive convenience, unstable across enumerations); `resolve_device` maps via `enumerate_devices()[n].endpoint.endpoint_guid` | number hits the index |

```text
resolve_device(device_ref) -> (device_guid, device_name, connection_name)
  1. device_ref starts with '{' -> use directly as GUID (format checked); name/connection matched
     from enumerate_devices (empty strings when not found)
  2. device_ref parses as number n -> enumerate_devices()[n] -> endpoint_guid / friendly_name /
     connection_name
  3. neither -> error "unknown device: <device_ref>" (with a list hint)
```

**`<file>` parameter (only `config set -f`)**: **source file** (content to write into the target
config.toml)

- Accepts **absolute** or **relative** paths (relative to the current working directory); quote
  paths with spaces (handled by the shell)
- The file must exist and be readable; the CLI reads it and writes the content **verbatim** to
  `C:\ProgramData\VxAPO\{GUID}\config.toml`
- Target path assembly: fixed `C:\ProgramData\VxAPO\{GUID}\config.toml` (v8.9 system-level
  CONFIG_ROOT — **not** the user-level Documents folder: audiodg's service account is
  `NT AUTHORITY\LOCAL SERVICE` (token carries the `NT SERVICE\Audiosrv` service SID; identity
  corrected 2026-10-02). The removed `documents_folder()` implementation (pre-v8.9) resolved the
  service account's Documents, which could not read files written by the CLI (a user process),
  causing "editing the Documents config has no effect"; ProgramData is shared by all users and
  shares the root with the snapshot directory). `{GUID}` is the device GUID (uppercase braced
  format); the directory is created when missing
- **The CLI does not modify file content** (pure copy); syntax validity is the driver's job
  (Lock/hot-reload parsing); the CLI only does file-level read-back

#### 4.4.2 Command table

| Command | Behavior | driver API |
|---------|----------|------------|
| `install -d <device> [--mode LfxGfx\|SfxMfx\|SfxEfx] [--no-child] [--verify] [--timeout=<sec>] [--progress-file=<path>]` | install (mode auto-detected when `--mode` is absent — EAPO three-tier via `detect_mode_for_guid`; `InstallConfig::default_config()` itself defaults to SfxEfx; `use_original=true` keeps the predecessor as child APO; `--verify` runs the verification loop: write → whole-service restart → pipe verification → scored retry, see 5.1b) | `write_install_config` (--verify) / `install_endpoint` (normal) |
| `uninstall -d <device>` | uninstall (including deleting the childApoPath key, v8.5) | `uninstall_endpoint` |
| `config set -d <device> -f <file>` | write `C:\ProgramData\VxAPO\{GUID}\config.toml` (v8.9/v9.11) | file write (the driver provides no config write — the CLI writes the file directly) |
| `config show -d <device>` | read back config.toml content (file-level verification) | file read-back |
| `config convert <old.txt> [out.toml]` | legacy EAPO-style txt → config.toml (GraphicEQ/Preamp/Wide/Aural/Reverb/Maximizer/Loudness) | file write |

### Phase C: snapshot = change listing + baseline preservation (added 2026-08-04) + registry dump retention

**The snapshot's real function = listing the changes (diff)**, not a pure backup.

> **Scope limitation (v8.8 scope correction)**: snapshots **capture registry changes only** —
> config file comparison is **not part of the CLI snapshot**; the **driver itself** detects config
> changes via directory watching + spec-fingerprint comparison (watcher → re-parse → compare
> against the active spec fingerprint).

1. **Snapshot content (`snapshot_device(guid)`)**: captures the device's **registry** state.
   Per the current implementation it stores the FxProperties 5 slot values
   (LFX/GFX/SFX/MFX/EFX GUIDs), the child-APO GUIDs (`childPreMix` / `childPostMix`), and the
   `childApoKeyExists` flag (the install info area under
   `HKLM\SOFTWARE\VxAPO\Child APOs\{guid}`). It does **not** include config files (config
   comparison belongs to the driver's spec fingerprint, see note above).
2. **Snapshot persistence**: stored at `%ProgramData%\VxAPO\snapshots\{guid}.json` (**the first
   snapshot = the pre-install baseline**)
3. **Change listing (`snapshot diff` / `changes`)**: `diff_snapshot(baseline, current)` compares
   the current registry state against the baseline and prints prefixed lines (the current CLI
   prints text prefixes; no ANSI colors):

   | Change | Display |
   |--------|---------|
   | added (not in baseline → present now) | `+ name = value` |
   | removed (in baseline → gone now) | `- name = value` |
   | modified (value differs) | `~ name = old → new` |
   | unchanged | `    name = value` |

   plus a summary line such as `Changes: +N added / -M deleted / ~K modified / S unchanged`.
4. **Baseline preservation semantics (explicit)**:
   - **Before install**: `snapshot_device` establishes the baseline (first snapshot)
   - **After install**: `snapshot diff` lists all registry changes made by this install (readable verification)
   - **At uninstall**: the comparison still uses **the very first baseline** (not a fresh snapshot
     before uninstall) — showing "did uninstall restore the original state / what was cleaned";
     **the snapshot is only replaced on the next reinstall** (re-snapshot before a new install,
     `snapshot_device(replace=true)`)
5. **Rollback**: `snapshot restore` returns to the baseline state (**registry only**). The driver
   transaction already guarantees automatic rollback on failure (registry level); the CLI snapshot
   adds **explicit restoration** (the user can restore manually). The current implementation
   restores by clearing the VxAPO slots via `uninstall_endpoint` (a subsequent `install` returns
   to the installed state).
6. **Keep the existing `[x]` registry dump + diagnostics**: regdump/reg helpers are kept (debug value)

### Phase D: P1 extensions (reserved, does not change the P0-7 main chain)

- `preset list/apply/set-intensity` (P1-2); EAPO config.txt migration is handled by
  `config convert` (implemented in v9.11)
- optional interactive wizard guiding toward the GUI App

---

## 4.5 GUID friendly names (added 2026-08-04)

The CLI's `knowledge.rs` maintains an **APO CLSID → product name mapping** so `list`/`status`
can show friendly names for slots occupied by EAPO/VxAPO CLSIDs (system APO property names have no
friendly names and would only display as `{d04e05a6...}`).

### 4.5.1 EAPO / VxAPO pre/postmix CLSIDs (source-confirmed)

| Product | Type | CLSID | Source |
|---------|------|-------|--------|
| **EAPO** | PreMix (pre_mix) | `{EACD2258-FCAC-4FF4-B36D-419E924A6D79}` | RegistryHelper.h 29 (source read) |
| **EAPO** | PostMix (post_mix) | `{EC1CC9CE-FAED-4822-828A-82A81A6F018F}` | RegistryHelper.h 31 (source read) |
| **VxAPO** | PreMix | `{41C34613-D391-459D-A039-72B2B15A1A1D}` | vx_reg_props.rs (v7.5 official GUID) |
| **VxAPO** | PostMix | `{B4A97313-ABC0-45ED-9C33-428B20D39428}` | vx_reg_props.rs (v7.5 official GUID) |

> **Implementation**: `knowledge.rs` keeps `KNOWN_APO_CLSIDS` (CLSID → "Equalizer APO PreMix" /
> "Equalizer APO PostMix" / "VxAPO PreMix" / "VxAPO PostMix"); `slot_friendly`/slot annotation
> consults this table first (the VxAPO keys are generated from the driver constants, so the
> literals cannot drift).

### 4.5.2 Premix / PostMix usage (second clarification)

| Concept | Definition | Position in the Windows audio chain | Typical slots (SfxEfx) | VxAPO usage |
|---------|-----------|-------------------------------------|------------------------|-------------|
| **Premix (pre-mix)** | APO processing **after the system mixer, at the start of the effect chain** | the engine runs Premix before mixing into other streams | **SFX(2)** (Win10+) / LFX(0) (legacy) | VxAPO PreMix instance: processes **independent streams/input sources** (e.g. a single app or capture); child delegation runs first, once per frame |
| **PostMix (post-mix)** | APO processing **after all streams are mixed, before endpoint output** | the engine runs PostMix after mixing all streams | **EFX(4)** (default) / MFX(3) (Bluetooth) / GFX(1) (legacy) | VxAPO PostMix instance: processes the **final mixed output** (system-wide EQ/effects) |

**Key distinctions**:
- **EAPO semantics**: the `preMix` flag is decided via `APOInitSystemEffects.APOInit.clsid`
  (EqualizerAPO.cpp 122); at install time `installPostMix = !input` (PostMix is installed on
  render devices only, DeviceAPOInfo.cpp 254).
- **VxAPO alignment**: the PreMix instance (`CLSID_VXAPO_PRE_MIX`) processes pre-mix streams; the
  PostMix instance (`CLSID_VXAPO_POST_MIX`) processes the final post-mix output; both share the
  same per-device config.toml, and child delegation runs first in both.
- **CLI display**: when annotating slot occupancy, `list`/`status` show "VxAPO PreMix/PostMix"
  and "EAPO PreMix/PostMix" so the user can quickly identify which APO took over which slot
  (paired with the v8.5 slot-loss reinstall prompt).

---

## 5. CLI behavior flows (added 2026-08-04 — install/verify/uninstall, down to functions, layered)

> **Layering**: CLI command (argument parsing/flow) → CLI helpers (resolve/snapshot/io) → driver
> API (the single install-layer write entry) → driver internals (Transaction/registry). Flows only
> go through the driver's public API and never touch driver internals.

### 5.1 Install flow (`vxapo-cli install -d <device> [--mode ...] [--no-child]`)

```text
main
 └─ cli::install(args)
     ├─ 1. require_admin()                                 # non-admin -> error exit (constraint 5)
     ├─ 2. (guid, name, conn) = resolve_device(args.device)  # 4.4.1: GUID or index -> triple
     ├─ 3. config = InstallConfig::default_config()
     │      config.install_mode = --mode parsed, else detect_mode_for_guid(guid)
     │        # explicit --mode wins; absent -> EAPO three-tier auto-detection
     │        # (InstallConfig::default_config() itself defaults to SfxEfx)
     │      config.use_original_apo_premix  = !args.no_child # default true: keep predecessor PreMix as child APO
     │      config.use_original_apo_postmix = !args.no_child # default true: keep predecessor PostMix as child APO
     ├─ 4. snapshot_device(guid, replace=true)               # Phase C: establish/replace baseline before install (registry only)
     │        #   persisted to %ProgramData%\VxAPO\snapshots\{guid}.json
     │        #   (config comparison is not part of the CLI snapshot — driver watching + spec fingerprint)
     ├─ 5. auto_register_driver()                            # refresh the global APO registration on every install (idempotent, completes the AudioEngine key)
     ├─ 6a. --verify -> install_verify (5.1b)                # write -> whole-service restart -> pipe verification -> scored retry
     ├─ 6b. else -> install_endpoint(&guid, &name, &conn, &config, true)
     │     └─ (driver internals, invisible to the CLI):
     │        internal: child_apo_key_exists(guid)           # -> full backup / non-full (lost-slot overwrite) path
     │        internal: transaction protection (reverse-order rollback on failure)
     │        internal: write FxProperties slots + childApoPath info area + processing-mode GUIDs
     │        internal: DisableProtectedAudioDG=1            # prerequisite for third-party APO loading
     │        internal: verify=true -> activate IAudioClient pipeline self-check (E3.4 v8.3: GetMixFormat + Initialize)
     │        internal: restart AudioSrv                     # new slot topology takes effect (EAPO install alignment)
     ├─ 7. success -> print "installed <guid> (mode <mode>, child APO keep=...)"; failure -> print driver Err + suggest snapshot
     └─ 8. ensure_default_config: if the per-device config.toml is missing, auto-import the
           exe-adjacent config.toml (else warn "APO will run without configuration")
```

> **`--verify` branch**: `--verify` does not use `install_endpoint`'s tail restart/self-check;
> it runs the 5.1b verification loop instead (write → whole-service restart → named-pipe
> verification → scored retry). The App's elevated path reads events incrementally via
> `--progress-file`; the stdout-direct path reads JSON line by line. The default config is
> ensured **before** verification so the verified state is the final usable state.

### 5.1b Verified install flow (`install --verify`, added 2026-08-19)

**Signature**: `install -d <device> [--mode ...] [--no-child] [--verify] [--timeout=<sec>] [--progress-file=<path>]`

**Event protocol**: one JSON object per line, written to both stdout and `--progress-file`
(append + flush):

| Event | Fields | Meaning |
|-------|--------|---------|
| `{"event":"phase","name":"admin\|resolve\|snapshot\|register\|config\|verify"}` | name | pre-verify phase marker (emitted by the install orchestration) |
| `{"event":"install_write","mode":"SFX_MFX"}` | mode | registry written (`write_install_config`, overwrite install; mode is uppercased) |
| `{"event":"service","action":"stopping\|stopped\|starting\|running"}` | action | whole-service restart phase (dependency-aware, polls STOPPED/RUNNING) |
| `{"event":"test","pipe":"VxAPODeviceTest","mode":"..."}` | pipe | pipe created + `DeviceTestPipeName` written, triggering starts |
| `{"event":"test","mode":"..."}` | mode | verification entered (user-facing events carry no scoring fields; scoring is CLI-internal for retry/complete only) |
| `{"event":"retry","from":"...","to":"...","reason":"..."}` | from/to | not full score → overwrite the next mode and retry |
| `{"event":"complete","success":true,"mode":"...","score":33,"attempts":1}` | success | success, exit code 0 |
| `{"event":"complete","success":false,"best_mode":"...","best_score":30,"attempts":3}` | best_* | all failed, keep the best-scoring configuration, exit code 1 |

(Diagnostic `trace` events may also appear; consumers should ignore unknown event types.)

**Flow**:

```text
install_verify(dev, config, timeout, progress_file)
 ├─ mode order = [preferred] + [sfx_efx, sfx_mfx, lfx_gfx] minus preferred (EAPO fallback order)
 ├─ per mode:
 │    ├─ write_install_config (pure registry write, transaction-protected)
 │    ├─ stop_audio_service_with_dependents(3) -> start_audio_service_with_dependents(5)
 │    ├─ create named pipe VxAPODeviceTest (DACL: SYSTEM + Administrators + Everyone)
 │    │   + write HKLM\SOFTWARE\VxAPO\DeviceTestPipeName
 │    ├─ trigger_apo_load: IMMDevice->IAudioClient->GetMixFormat->Initialize(shared, 100ms)
 │    │   (Initialize only, no stream start, aligned with EAPO; E_PENDING/DEVICE_INVALIDATED
 │    │   retried 5x500ms; no trigger-level timeout — slow devices are not cut off; the 20s
 │    │   global watchdog is the backstop)
 │    ├─ pipe collection <=2s (fixed 200ms x 10 iterations): the driver DLL's Initialize reports
 │    │   stage JSON back through the pipe
 │    ├─ scoring: premix_init 20 / postmix_init 10 / child_premix 2 / child_postmix 1
 │    │   full score render=33, capture=22; child criterion = registry expectation exists AND a
 │    │   child_apo message arrived (an empty expectation also passes — a clean install with no
 │    │   original APO still reaches full score)
 │    └─ cleanup: delete DeviceTestPipeName + close the pipe (always, in finally)
 ├─ score == max -> complete success:true, exit 0
 │    (no extra endpoint restart — the service restart already rebuilt the endpoint graph,
 │    avoiding a second audio interruption during install)
 └─ all failed -> uninstall_endpoint rolls the registry back + start_audio_service_with_dependents
      ensures the service runs -> complete success:false, exit 1
```

**Timeouts**: global watchdog 20s (any hang is force-terminated); phase budgets: service stop 3s,
start 5s, pipe collection 2s, trigger retries limited to E_PENDING/DEVICE_INVALIDATED 5x500ms (no
outer timeout, so slow devices are not killed prematurely). `--timeout` defaults to 180s (the
overall install budget).

### 5.2 Verification flows (after `install` → `config set` + `config show` + `status`)

```text
vxapo-cli config set -d <device> -f <file>
 └─ cli::config_set(args)
     ├─ 1. (guid, ..) = resolve_device(args.device)
     ├─ 2. path = C:\ProgramData\VxAPO\{guid}\config.toml   # v8.9 system-level CONFIG_ROOT (shared by audiodg/SYSTEM and user processes)
     ├─ 3. fs::create_dir_all(parent) + read source file args.file, write verbatim to path (pure copy, no rewriting)
     └─ 4. self-check after write: `config show` read-back (file level); DSP validity is the driver's job

vxapo-cli config show -d <device>
 └─ cli::config_show(args)
     ├─ 1. resolve_device -> guid
     ├─ 2. path = C:\ProgramData\VxAPO\{guid}\config.toml   # v8.9 system-level CONFIG_ROOT
     ├─ 3. fs::read_to_string(path) -> print content
     └─ 4. print config.toml content (file-level read-back verification, no DSP parsing)

vxapo-cli status / list
 └─ cli::status() / cli::list()
     ├─ 1. devices = enumerate_devices()                        # driver: the single enumeration entry
     ├─ 2. print per device: endpoint name / installed_version / install_mode / 5-slot occupancy
     │      (knowledge::KNOWN_APO_CLSIDS friendly names "VxAPO PreMix/PostMix", "EAPO PreMix/PostMix")
     │      / is_disabled / is_unplugged
     └─ 3. slot-loss annotation (v8.5): installation-mode premix/postmix slot ≠ VxAPO CLSID ->
            "⚠ taken over by <friendly name>, reinstall needed"
            (triggers install reinstall -> install_endpoint internally overwrites the backed-up
             child APO with the latest predecessor)
```

### 5.3 Uninstall flow (`vxapo-cli uninstall -d <device>`)

```text
main
 └─ cli::uninstall(args)
     ├─ 1. require_admin()
     ├─ 2. (guid, ..) = resolve_device(args.device)
     ├─ 2b. **stale fallback**: find_endpoint_path(guid) fails AND guid is in `stale list`
     │         -> go directly to stale_cleanup(guid), clearing the Child APOs key + config directory, and return
     ├─ 3. **no fresh snapshot** (baseline preservation, explicit) — diff_snapshot(baseline, current)
     │        uses the very first baseline
     │        # shows "did uninstall restore the original state / what was cleaned"; the baseline is
     │        # only replaced at the next reinstall (snapshot_device(replace=true))
     ├─ 3b. **pre-uninstall service stop (CLI layer)**: stop_audio_service ->
     │        audiodg::wait_for_audiodg_exit(5000) **event-driven wait**
     │        (audiodg exits with the audio service; the wait releases the vxapo_driver.dll module
     │         image — see step 4 — it is not about "unlocking slot values")
     ├─ 4. uninstall_endpoint(&guid)                            # driver: deletes the FxProperties slot VxAPO CLSIDs -> restarts endpoint/service
     │     └─ (driver internals) stopping AudioSrv is not "to delete values" (deleting values only
     │          needs a KEY_SET_VALUE handle; measured to succeed even with the DLL loaded and
     │          audiodg holding the endpoint) but to ① release the vxapo_driver.dll module image
     │          (otherwise a later reinstall/DLL swap cannot overwrite it; the NSIS installer hooks
     │          stop the service for the same reason) ② let the endpoint restart rebuild the APO
     │          chain immediately
     │     └─ (driver internals) deletes the childApoPath\{guid} key entirely (v8.5 uninstall always
     │          deletes — the next install takes the full path again)
     │     └─ (driver internals) restarts AudioSrv (restore output)
     ├─ 5. success -> print "uninstalled <guid>" + snapshot diff change summary
     ├─ 6. failure -> print driver Err (step-level error -> 5.4) + suggest snapshot restore
     └─ 7. (the current implementation performs no automatic config.toml cleanup; config files are kept)
```

### 5.4 Command state flow (added 2026-08-04)

> **Goal**: make explicit what each command must check, which states an execution can move the
> device into, which commands are allowed in each state, and the transition logic — every step
> absolutely strict + readable feedback + error checking (requirement).

#### 5.4.1 State definitions (CLI device lifecycle states)

| State | Criterion (`DeviceInfo` + snapshot existence) | Meaning |
|-------|-----------------------------------------------|---------|
| **U**nmanaged | `!is_installed()` && no snapshot | device never managed by VxAPO |
| **B**aselined | `!is_installed()` && snapshot exists (baseline) | installed once then uninstalled / baseline only; slots currently non-VxAPO |
| **I**nstalled | `is_installed()` && snapshot exists (baseline) | VxAPO owns the slots + pre-install baseline exists |
| **L**ost | `is_installed()` (version non-empty) but **installation-mode slots ≠ VxAPO CLSID** | slots taken over by a third party (v8.5 loss) |

> no snapshot + `is_installed()` = anomaly (snapshot file deleted) — treated as `L` with a
> "snapshot missing, reinstall will rebuild the baseline" hint.

#### 5.4.2 Command × state matrix (commands allowed per state)

| Command | U | B | I | L | Preconditions (else reject + readable error) |
|---------|---|---|---|---|-----------------------------------------------|
| `list` / `status` | ✅ | ✅ | ✅ | ✅ | none (pure read-only) |
| `config show` | ✅ | ✅ | ✅ | ✅ | target config exists (else "not configured" hint) |
| `snapshot diff` | ✅ | ✅ | ✅ | ✅ | snapshot exists (else "no baseline, install first") |
| `install` | ✅ | ✅ | ✅ | ✅ | ① admin ② resolve_device ok ③ if `is_installed()`, confirm "reinstall replaces the baseline" |
| `config set` | ✅* | ✅* | ✅ | ✅ | ① target dir creatable ② source file exists/readable ③ **if not I → warn "device not installed, config will not take effect" but proceed** (*U/B allowed with warning) |
| `uninstall` | ❌ | ✅ | ✅ | ✅ | ① admin ② snapshot exists (else reject "no baseline to compare") ③ `is_installed()` or L |
| `snapshot restore` | ❌ | ✅ | ✅ | ✅ | snapshot exists (else reject); verify with `status` afterwards |

#### 5.4.3 State transitions (after command execution)

```text
U ── install (establish baseline + install) ──────────> I
U ── snapshot diff (no baseline) -> reject (error) -> still U
B ── install (replace baseline + install) ────────────> I
B ── uninstall (nothing to uninstall) -> reject -> still B
I ── slot overwritten by third party ─────────────────> L (detected by status: installation-mode slots ≠ VxAPO CLSID)
I ── uninstall (clear slots + delete childApoPath key; baseline preserved) -> B
L ── install (lost-slot reinstall: overwrite backed-up child APO with latest predecessor) -> I
L ── uninstall (clear slots; baseline preserved) ─────> B
any ── reinstall while installed -> replace baseline + reinstall -> I
```

**Absolutely strict error checking per step (requirement)**: no command failure may be silent —

1. driver `Err` ≠ empty → print **error code + message + the step it belongs to** (e.g.
   `install_endpoint step 4 (write child APO config) failed: ...`)
2. assert preconditions before each step (see the 5.4.2 "preconditions" column); on failure print
   "rejected: <reason>" and **do not proceed**
3. on success print explicit feedback (e.g. `✓ installed <guid>`,
   `✓ config written (<N> bytes)`), on failure `✗ <error>`
4. after every `snapshot diff`, print a summary line:
   `Changes: +N added / -M deleted / ~K modified / S unchanged` (readability)
5. when a write operation (install/uninstall/config set/snapshot restore) fails, hint the
   recovery path (snapshot restore / retry)

### 5.5 End-to-end verification index (P0-7 DoD)

```text
① install -> status                    # verify FxProperties slots taken by VxAPO CLSID + childApoPath key created
② install -> config set -> config show  # verify config write + syntax read-back
③ install -> config set -> play audio  # P0-4 manual listening verification (hot reload works, no pops)
④ install (use_original=true) -> play  # P0-6 child delegation chain end-to-end (real audiodg + registered APO)
⑤ uninstall -> status                  # verify slots restored/cleared + childApoPath key deleted + next install takes the full path
⑥ slot-loss simulation: manually set a slot to non-VxAPO -> status annotates loss -> install reinstall -> child APO overwritten with latest predecessor (P0-6 ④)
```

### 5.6 Stale GUID flow (`stale`, added 2026-09-10)

**Signatures**:

```text
vxapo-cli stale list [--json]
vxapo-cli stale migrate --from <oldGuid> --to <newGuid> [--config-from <guid>] [--snapshot-from <guid>] [--json]
vxapo-cli stale cleanup -d <guid> [--json]
vxapo-cli stale fix-acl -d <guid> [--json]
```

```text
stale list (read-only, no elevation)
 └─ driver install::device::stale::list_stale_installs()
     scans HKLM\SOFTWARE\VxAPO\Child APOs\{oldGuid} + C:\ProgramData\VxAPO\{oldGuid}
     + snapshots\{oldGuid}.json, matches active endpoints by **layered identity** (v9.24:
     endpoint history property -> old endpoint key instance ID -> stored record identity ->
     hardware ID + product name unique hit; ambiguous/missed stays unmatched — no guessing)
     -> emits StaleInstall[] (target_state = matched_healthy / matched_partial / unmatched,
        matched_by = endpoint_history / device_instance_id / stored_identity / hardware_id / null)

stale migrate (admin)
 └─ 1. require_admin
    2. **no service stop** (measured 2026-09-16: writing/deleting FxProperties values only needs
       a KEY_SET_VALUE handle; audiodg holding the endpoint does not block it — stopping the
       service was a historical misconception)
    3. migrate_install(from, to, config_from, snapshot_from)
         · default sources: old records of the same device instance vs the target's existing files
           -> latest config, earliest snapshot
         · migrate config.toml + snapshot -> repair the new GUID install state -> delete old GUID records
         · refresh the target record identity and write the EndpointHistory union back (including
           the new GUID, so the next GUID change can be recognized)
         · back up existing target files to C:\ProgramData\VxAPO\_migration_backup\{guid}\
         · if the repair branch rewrote slots, the driver calls restart_endpoint_device afterwards
           so the slot changes take effect (the engine caches the endpoint APO chain)
    4. emit MigrationReport (success/config_migrated/snapshot_migrated/install_repaired/
       removed_guids/warnings); ensure audiosrv is running at the end (idempotent)

stale cleanup (admin)
 └─ cleanup_orphan(guid): remove old records that match no active endpoint

stale fix-acl (admin)
 └─ fix_config_acl(guid): grant the interactive user Modify on config/snapshot
    (icacls SID *S-1-5-4)
```

> Migration runs through the elevated CLI, so config.toml may inherit an administrator ACL; when
> the App's first write is denied it automatically calls `stale fix-acl` and retries (see
> App Reference Specification §4.4).

---

## 6. Modification constraints (hard)

1. **Never touch pipeline/RT**: the CLI depends only on `install/` + file-level config read-back +
   `sys/` (through the driver); violating the three-layer separation in
   `overview/Project Overview.md` is forbidden.
2. **Never write the registry directly**: install/uninstall always go through
   `install_endpoint`/`uninstall_endpoint` (transaction protection); slot-loss detection is
   read-only (`enumerate_devices` + slot values); writes go through the driver.
3. **Never duplicate implementations**: endpoint enumeration uses `enumerate_devices` (the single
   entry, install 5.4); the CLI's self-implemented winreg enumeration is deprecated for command
   flows (kept only for viewer mode).
4. **Config writes belong to the CLI**: the driver only falls back to a default config
   (object 7.1.8); the CLI owns `config set` (intent: "the application layer writes files").
5. **Privileges**: install/uninstall require administrator (HKLM writes); the CLI detects non-admin
   execution and refuses with a hint.
6. **Stale migration is an install-layer write**: `stale migrate/cleanup/fix-acl` require
   administrator and only go through the APIs of `install::device::stale` +
   `install::selector::operation`; never write the registry directly.

---

## 7. Relations

- P0-7 (roadmap): CLI end-to-end verification — once this specification lands, P0-7 can enter Implementing
- P0-6 legacy: complete child-delegation chain testing + P0-4 listening verification — both covered by CLI (install/config set) end-to-end testing
- `overview/Project Overview.md` §5 (three-layer separation) / §7 (slot-loss detection v8.5) / §11 (config.toml model) — superior constraints on CLI behavior

