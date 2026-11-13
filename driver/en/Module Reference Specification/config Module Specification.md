# config Module Specification

**Boundary**: Does not know `install/` or `object/`. Responsible only for parsing the configuration file and building the Filter chain. It does **not** directly operate `Chain`.

**Allowed dependencies**:
- `sys/`
- `utils/`
- `pipeline/dsp/filter.rs` (`Filter` trait + `DspContext`)
- `pipeline/dsp/model.rs` (`ChainModel` / `EffectType`, the v9.11 conversion target)
- `pipeline/dsp/factory.rs` (`create_from_model` static dispatch, v9.11)
- (channel names are injected by the caller into `DspContext.channel_names`; config does **not** directly depend on `sys/audio_defs`)

**Forbidden dependencies**:
- `install/`, `object/`
- `pipeline/process.rs`, `pipeline/chain.rs`, `pipeline/context.rs`
- **any concrete `pipeline/dsp/*.rs` implementation file (e.g. `peq_hybrid.rs`, `wide.rs`)**

## Module tree

```
config/
├── parser.rs              # TOML config parsing (FileModel -> ChainModel -> chain)
├── model.rs               # FileModel (serde, includes name/group/meta APP metadata)
├── error.rs               # ConfigError
└── watcher.rs             # config file change watcher
```

## 6.0 v9.11 TOML config model (current, replaces the old line commands)

> Since v9.11 the config file is `config.toml` (per-device
> `C:\ProgramData\VxAPO\{GUID}\config.toml`). The old `config.txt` line-command system
> (`config/commands/*`, EAPO alignment, `GraphicEQ:` etc.) has been removed entirely;
> the migration tool `vxapo-cli config convert` performs a one-time conversion.

**Two-layer model (final)**:
- `FileModel` (`config/model.rs`): TOML deserialization target with `version`, top-level
  `enabled` (v9.18 master switch; `false` = whole-chain passthrough), `[meta]`, and
  `[[effects]]` (`name`/`group` are APP metadata);
- `ChainModel` (`pipeline/dsp/model.rs`): pure DSP semantic model (type, params,
  `enabled`, `channels`), no APP metadata;
- The conversion (`FileModel -> ChainModel`, with range/band-count/channel-name
  validation) is the **config layer's responsibility**; the dependency direction stays
  `config -> pipeline/dsp`.

**Schema (v1)**:

```toml
version = 1

[meta]
app = "vxapo"
schema = 1

[[effects]]
type = "peq"
name = "footstep boost"          # optional, APP metadata, ignored by the driver
group = "FPS preset"             # optional, APP metadata, ignored by the driver
crossover_hz = 200
[[effects.bands]]
fc = 1000
gain_db = 3.0
q = 1.0
```

**Effect types (v1 retained set)**: `peq` (hybrid, see pipeline 4.22), `preamp`,
`aural`, `reverb`, `compressor`, `wide`, `loudness`; `enabled = false` bypasses the
effect; `channels = ["FL", "FR"]` restricts the affected channels (default: all).
Legacy `maximizer` / `leveler` types and their old parameter keys are accepted but
ignored, mapping to `compressor` defaults.

**Validation**: unknown type / unknown key / inapplicable field / missing required key /
out-of-range value / PEQ band count outside [1, 31] / duplicate channel names or channel
names not present on the device → the whole file fails to parse and the old chain is kept.

> **The band-count cap is per block (v9.x reality — do not write "global total <= 31")**:
> 1-31 bands per block; blocks **without declared channels** share one 31-band budget
> (`unscoped 'peq' bands count 32 exceeds max 31`); blocks **with declared channels** get
> 31 bands per channel (`channel 'L' peq bands count 32 exceeds max 31`) — so
> `L 20 + R 20` is legal and the cross-channel total may exceed 31.
> Basis: `config/model/tests.rs` — `total_peq_band_cap_enforced` (unscoped blocks share
> the budget), `per_channel_peq_band_cap` (per-channel 31; `L 20 + R 20` passes, 32 on one
> channel reports `channel 'L' peq bands count 32 exceeds max 31`),
> `band_count_out_of_range_rejected` (32 bands in one block → `out of range [1, 31]`),
> `single_band_peq_accepted` (lower bound 1). The App side (`addBand` / `applyPreset`)
> validates with the same rule: without the channel selector the count is per device;
> with it, per selected channel.

**Fingerprint and hot reload**: the spec fingerprint = `EffectConfig::spec()` (stable
serialization of DSP fields; `name`/`group`/`meta` do not participate); the watcher
monitors `config.toml`.

## Reference dependency table

> v9.17 (single source of truth): this table is no longer maintained independently — the
> main specification `Module Reference Specification (Without Detailed Modules).md`,
> section 11 "Reference dependency table", is the only baseline; the per-file allowed and
> forbidden dependencies of config are listed line by line there.

## 6.0 `config/error.rs`

**Responsibility**: the dedicated error type for config parsing.

**References**: no external dependencies.

**Exports to**: all `config/` submodules.

**Public API**:

```rust
#[derive(Debug, Clone, thiserror::Error)]
pub enum ConfigError {
    /// File I/O error.
    #[error("I/O error reading {path}: {message}")]
    IoError { path: String, message: String },
    /// Command syntax error (file name and line number included).
    #[error("Syntax error in {file}:{line}: {message}")]
    SyntaxError { file: String, line: usize, message: String },
    /// TOML deserialization failure (v9.11).
    #[error("TOML error in {file}: {message}")]
    TomlError { file: String, message: String },
    /// FileModel -> ChainModel conversion/validation failure (v9.11).
    #[error("Config model error in {file}: {message}")]
    ModelError { file: String, message: String },
    /// AbortFile termination (set by the Device: command).
    #[error("Parse aborted by Device: AbortFile")]
    AbortFile,
}

impl From<ConfigError> for crate::utils::vx_error::VxApoError { /* -> VxApoError::config */ }
```

> Variant names and Display strings are the external error-message contract; changing them
> requires synchronizing the CLI/App texts. `ConditionFalse` is not returned as an error —
> when a condition is not met the line is simply skipped, without affecting parsing.

## 6.1 `config/parser.rs`

**Responsibility**: config file parser (v9.11 TOML). Reads `config.toml` →
`toml::from_str::<FileModel>` → `to_chain_model` (validation) → `factory::create_from_model`
to build the Filter chain, and produces the spec fingerprint (`EffectConfig::spec()`
series — used by the object layer for hot-reload change detection).

**References**:
- `crate::config::error::ConfigError`
- `crate::config::model::FileModel`
- `crate::pipeline::dsp::model::{ChainModel, EffectType}`
- `crate::pipeline::dsp::filter::{Filter, DspContext}`
- `crate::pipeline::dsp::factory::create_from_model`
- (channel names are injected via `DspContext.channel_names`; config does not directly depend on `sys/audio_defs`)

**Exports to**: `object/apo.rs` (Lock / hot reload).

### ConfigParser

> **The following API list is the pre-v9.11 text-command implementation (all removed)**:
> `normalize_tokens`, `normalize_number`, `split_command_value`, `produce_spec`,
> `is_known_dsp_command` and the `XxxFactory` dynamic registry — **none of these symbols
> exist any more** (0 hits repo-wide). The current parser API is in `config/parser.rs`:
> `ConfigParser::parse_file_with_spec(path, ctx) -> (Vec<Box<dyn Filter>>, SpecChain)`,
> with the flow `toml::from_str::<FileModel>` → `FileModel::to_chain_model` →
> `factory::create_from_model`. The v7.9 / v7.11 / v8.3 history and EAPO comparison notes
> kept below are **historical rationale** and still valid; the conversion entry point for
> the old text format lives in the CLI (`vxapo-cli/src/commands/convert.rs`, EAPO txt → TOML).

```rust
/// Spec fingerprint (v9.11): stable serialization of `EffectConfig::spec()`
/// (DSP fields; `name`/`group`/`meta` do not participate). The ordered spec chain is
/// compared item by item for hot-reload decisions.
pub type FilterSpec = String;

/// Spec fingerprint (spec chain): the ordered sequence of filter_specs produced by one
/// full parse (the `[[effects]]` array order is the processing order).
pub type SpecChain = Vec<FilterSpec>;

/// Config file size gate (thread IO safety cap).
pub const MAX_CONFIG_FILE_SIZE: u64 = 128 * 1024;

/// TOML config parser (stateless; model validation lives in config/model.rs).
pub struct ConfigParser;

impl ConfigParser {
    pub fn new() -> Self;

    /// Parses the config file and produces the spec fingerprint.
    /// Missing file = passthrough (empty chain, v9.12); syntax/model validation failure
    /// → whole-file failure (callers degrade per scenario: Lock → passthrough,
    /// hot reload → keep old chain). Files above MAX_CONFIG_FILE_SIZE (128 KiB) → IoError.
    pub fn parse_file_with_spec(&self, path: &str, ctx: &DspContext)
        -> Result<(Vec<Box<dyn Filter>>, SpecChain), ConfigError>;

    /// Parses a config string (TOML). Test-only (`#[cfg(test)]`).
    pub fn parse_string(&self, content: &str, ctx: &DspContext)
        -> Result<Vec<Box<dyn Filter>>, ConfigError>;

    /// Parses a line list (compatibility entry: joined with newlines, then TOML parsing).
    /// Test-only (`#[cfg(test)]`).
    pub fn parse_lines(&self, lines: &[String], ctx: &DspContext)
        -> Result<Vec<Box<dyn Filter>>, ConfigError>;
}

/// Reads the config file content (UTF-8 first; skips a UTF-8 BOM; lossy degradation
/// for non-UTF-8 bytes).
pub fn read_config_file(path: &Path) -> Result<String, ConfigError>;
```

**Chain construction** (`build_chain`): validates per-effect `channels` against
`DspContext.channel_names` (unknown channel → `ModelError` "not present in device
channels"), wraps scoped effects in `ChannelScopedFilter`, and **merges adjacent,
same-channel-scope, enabled `peq` blocks into a single hybrid FIR** (31 cascaded sections
as N independent convolutions would scale RT cost linearly; merged they share one FIR with
the summed dB response and a single minimum-phase).

### filter_spec production contract (v7.9, P0-4; since v9.11 `EffectConfig::spec()`)

**Principle**: the spec is a **text fingerprint**, not "DSP semantic equivalence" — since
v9.11 it is produced uniformly by the model layer's `EffectConfig::spec()`
(type/enabled/channels/all parameters, `{:.6}` fixed precision); `name`/`group`/`meta`
do not participate (APP metadata changes do not trigger hot reload).

> **Unit-token boundary (historical)**: token normalization only applied `normalize_number`
> to tokens parseable as f64 and **did not strip units** (`-6.0 dB` → `-6\x1FdB` ≠ `-6`).
> Unit tokens are DSP knowledge; stripping them would violate "the spec is produced by the
> parser with zero DSP knowledge". A unit difference (`-6.0 dB` vs `-6.0`) triggered one
> transition — manual file edits are an edge case, the transition uses a raised cosine and
> is inaudible (overview/Project Overview.md).

**Bare-command reachability reversal (v7.11, execution-side issue ② → product decision)**:
> Lines without a colon must not be parsed (overview/Project Overview.md, "syntax strictness": the
> string before the colon decides the parse target and must be a strict keyword).
> **EAPO behavior fact (v8.3 note, S5)**: EAPO silently skips colon-less lines
> (FilterEngine.cpp 329-330: `pos = line.find(':')`; when `pos == -1` the whole line is
> not parsed and no error is raised) — VxAPO "reject with an error" is **stricter than
> EAPO**, an **intentional difference** (intent: strict input guarantees parse feedback).
> Therefore the v7.9 bare-command reachability fix was deleted: `split_command_value`
> with zero colons → `SyntaxError` "missing colon" (whole-file failure); the
> `produce_spec(cmd, "")` branch remained only as defense.

### ParseContext (mutable state during parsing) — deprecated

> **Deprecated in v9.11**: the ParseContext / line-by-line dispatch / command-handler
> details below describe the pre-v9.11 txt command system, which has been deleted; kept as
> historical reference, not the current implementation.

```rust
pub struct ParseContext<'a> {
    pub filters: &'a mut Vec<Box<dyn Filter>>,
    pub registry: &'a FilterRegistry,
    pub dsp_ctx: &'a DspContext,
    pub stage: ParseStage,
    pub is_capture: bool,
    pub current_file: PathBuf,   // v7.4: &'a Path -> owned PathBuf (Include sub-parses)
    pub line_number: usize,
    pub abort_file: bool,
    pub cond_stack: CondStack,
    pub variables: Variables,
    pub include_depth: usize,
    pub current_channels: Vec<String>,
    pub all_channels: Vec<String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ParseStage { None, PreMix, PostMix, Capture }
```

### Parse flow (historical)

```rust
/// Splits `Command: value` (v7.11 strictness — single-colon check).
/// Exactly one colon: Ok((command, value)); zero colons: Err (rejected — intentional
/// difference vs EAPO's silent skip); more than one colon: Err ("extra colon").
fn split_command_value(line: &str) -> Result<(&str, &str), ConfigError>;
```

The line-by-line dispatch loop handled condition commands first (`If`/`ElseIf`/`Else`/
`EndIf`, always, to manage the stack), skipped non-condition commands while in a false
branch, then dispatched `Device`/`Stage`/`Channel`/`Eval`/`Include`, the REW dynamic
prefix `Filter N:` (`starts_with("filter ")`), and finally whitelist-checked DSP commands
(`is_known_dsp_command` against `registry.factory_names()`); `VSTPlugin` was special-cased
to a clear "not enabled (reserved)" error; `Unmatched` was narrowed to "known command with
invalid arguments". An unbalanced condition stack at EOF → "unterminated If: block".

### Command keyword whitelist (v7.12 — fix of the "unknown command" detection failure)

**Root cause**: the v7.11 "Unmatched → SyntaxError" failed — `BogusCommand: x` had a colon
and a non-empty value, fell into `try_create(x)`, and was absorbed by Convolution's lenient
parsing (any non-empty string = IR path) → created successfully → Unmatched was never
reached → "unknown command" never triggered.

**Decision**: every valid DSP command keyword must be known to the parser — three-stage
flow: `split_command_value` (colon-count strictness) → command name ∈ whitelist? (no →
SyntaxError "unknown command", not registered) → static match / REW prefix / `try_create`
(Unmatched narrowed to "invalid arguments for a known command").

| Category | Command names | Hit path |
|----------|---------------|----------|
| Static commands | `Device`/`Stage`/`Channel`/`Eval`/`Include`/`Filter`/`GraphicEQ`/`Preamp`/`Copy`/`Delay` | static `match` (condition commands handled earlier in the stack stage) |
| REW dynamic prefix | `Filter N:` (`Filter 1:`/`Filter 12:`) | guard branch `starts_with("filter ")` |
| registry command names | `IIR`/`Biquad`/`Convolution`/`VSTPlugin`/`LoudnessCorrection` (=`registry.factory_names()`, pipeline 4.10) | `_` branch `is_known_dsp_command` |

## 6.2 `config/watcher.rs`

**Responsibility**: config file change monitoring (**Win32 event-driven**, v7.8 revision —
aligned with EAPO's `notificationThread`). **No polling** (the old 2000 ms polling had high
latency and wasted CPU).

**References**:
- `crate::config::error::ConfigError`
- `crate::utils::vx_error::VxApoError`
- `windows::Win32::Storage::FileSystem::{FindFirstChangeNotificationW, FindNextChangeNotification, FindCloseChangeNotification}`
- `windows::Win32::System::Threading::{WaitForMultipleObjects, CreateEventW, SetEvent}`
- `windows::Win32::Foundation::{HANDLE, WAIT_OBJECT_0}`

**Exports to**: `object/apo.rs`.

**Public API**:

```rust
/// Change events reported by the watcher.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum WatchEvent {
    /// A change occurred inside the watched directory (**directory-level notification** —
    /// `FindFirstChangeNotificationW` provides no file name, so per-file filtering is
    /// impossible). The trigger (object/apo.rs hot_reload) re-parses config.toml and uses
    /// the spec fingerprint comparison to decide whether to really switch (unchanged
    /// content → idempotent skip, no audible side effect).
    DirectoryChanged(PathBuf),    // watch_dir
    RegistryChanged,              // kept: poll_registry hash fallback (low frequency)
}

/// Config directory change watcher (v7.8/v7.9/v7.10 event-driven — externally driven).
///
/// Watches a **directory** (not the file — a file handle goes stale when the file is
/// deleted and recreated; a directory is naturally robust).
/// **Thread model (v7.10)**: `ConfigWatcher` does **not** spawn its own thread — it only
/// manages the notification handle + blocking wait API. The thread is created and driven
/// by the caller (`object/apo.rs::start_watcher`, 7.1.9): loop `wait_and_handle()`;
/// `DirectoryChanged` triggers `hot_reload` (7.1.18). Dedup window 10 ms (aligned with
/// EAPO: after FindNextChangeNotification, WaitFor(1, 10 ms) merges the editor's
/// "write temp file + rename" notifications). Exit: shutdown_event set →
/// `wait_and_handle` returns false → the caller's thread exits (SetEvent + join at
/// UnlockForProcess; v7.9 lifetime follows the lock period).
pub struct ConfigWatcher { ... }

impl ConfigWatcher {
    /// Creates the watcher (**does not start a thread**) — handle creation + state init.
    /// - watch_dir: the watched **directory** (e.g. `C:\ProgramData\VxAPO\{GUID}`),
    ///   not the config.toml file itself
    /// - shutdown_event: externally owned exit event (held by the APO instance;
    ///   SetEvent at UnlockForProcess → wait_and_handle returns false → the caller joins
    ///   its thread)
    /// If `FindFirstChangeNotificationW` fails (e.g. directory missing), the notify handle
    /// is invalid and `wait_and_handle` immediately returns false (equivalent to no watch).
    pub fn new(watch_dir: PathBuf, shutdown_event: HANDLE) -> Self;

    /// The notification handle (for caller-side validity checks).
    pub fn notify_handle(&self) -> HANDLE;

    /// Waits for and processes one event (blocks until a change or shutdown).
    /// Returns true: directory changed (notify handle reset, may wait again).
    /// Returns false: shutdown set, invalid handle, or wait failure.
    /// If `FindNextChangeNotification` fails, the handle is rebuilt once (self-healing);
    /// only if the rebuild also fails does it return false — never spins.
    pub fn wait_and_handle(&mut self) -> bool;

    /// Checks registry changes (hash comparison, low frequency). Test-only (`#[cfg(test)]`).
    pub fn poll_registry(&mut self, current_hash: u64) -> Option<WatchEvent>;

    /// Stops watching and releases the handle. Test-only (`#[cfg(test)]`); in production
    /// the APO object sets the shutdown event and drops the watcher, whose `Drop` closes
    /// the notification handle (idempotent).
    pub fn shutdown(&mut self);
}
```

> **Directory-level semantics (v7.9)**: `FindFirstChangeNotificationW` is a
> directory-level notification — it only says "something changed under the watched
> directory" and provides **no file name** (file-name information requires
> `ReadDirectoryChangesW`). Therefore `WatchEvent::ConfigFileChanged(PathBuf)` /
> file-name filtering (old 6.2) is impossible and was removed in v7.9, unified into
> `DirectoryChanged(watch_dir)`. The object layer's `hot_reload` (7.1.18) responds to any
> directory change: 128 KB gate → re-parse → spec fingerprint comparison (item by item
> against active_spec) — unchanged content (unrelated files/comments/rewrites) is skipped
> idempotently.
>
> **Lifecycle (v7.9)**: `ConfigWatcher` is held by `ApoObject.watcher: Option<ConfigWatcher>`,
> created at the end of `LockForProcess` (7.1.9), and at `UnlockForProcess` the shutdown
> event is set and the thread joined (7.1.10). While unlocked there is no audio stream, so
> hot reload is meaningless; a re-Lock necessarily re-reads the config and establishes a new
> baseline.
>
> **Old polling mode removed (v7.8)**: `poll()` / `should_poll()` / `poll_interval_ms` /
> `Deduplicator(500ms)` removed — event-driven + 10 ms dedup kept the same rain-prevention
> semantics with latency down from ~2000 ms to ~10 ms and zero idle CPU.

## 6.3 Historical command system (removed)

Since v9.11, the `config/commands/*` line-command system has been removed entirely.
Parsing now uses only the TOML model; effects are constructed by `pipeline/dsp/factory.rs`
static dispatch. The old command syntax, `GraphicEQ:`, `Channel:` and other line commands
are no longer a runtime format; migration is done by `vxapo-cli config convert`.

For detailed effect parameters and DSP behavior, see `driver/zh/配置与DSP设计.md`
(English: `driver/en/Configuration and DSP Design.md`).
