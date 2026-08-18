# config Module Specification

**Boundary**: Does not know `install/` or `object/`. Responsible only for parsing the configuration file and building the Filter chain. It does **not** directly operate `Chain`.

**Allowed dependencies**:
- `sys/`
- `utils/`
- `pipeline/dsp/filter.rs` (`Filter` trait + `DspContext`)
- `pipeline/dsp/model.rs` (`ChainModel` / `EffectType`)
- `pipeline/dsp/factory.rs` (`create_from_model` static dispatch)

**Forbidden dependencies**:
- `install/`, `object/`
- `pipeline/process.rs`, `pipeline/chain.rs`, `pipeline/context.rs`
- any concrete `pipeline/dsp/*.rs` implementation file

## Module tree

```
config/
├── parser.rs              # TOML config parsing (FileModel -> ChainModel -> chain)
├── model.rs               # FileModel (serde, includes name/group/meta APP metadata)
├── error.rs               # ConfigError
└── watcher.rs             # config file change watcher
```

## 6.0 v9.11 TOML config model

Since v9.11 the config file is `config.toml` (per-device `C:\ProgramData\VxAPO\{GUID}\config.toml`). The old `config.txt` line-command system (`config/commands/*`, EAPO alignment, `GraphicEQ:` etc.) has been removed; `vxapo-cli config convert` performs one-time migration.

Two-layer model:
- `FileModel` (`config/model.rs`): TOML deserialization target with `version`, top-level `enabled` (v9.18 master switch; false = whole-chain passthrough), `[meta]`, and `[[effects]]` (`name`/`group` are APP metadata).
- `ChainModel` (`pipeline/dsp/model.rs`): pure DSP semantic model (type, params, `enabled`, `channels`), no APP metadata.
- Conversion (`FileModel -> ChainModel`) including range/band-count/channel-name validation is the config layer's responsibility.

Example schema:

```toml
version = 1

[meta]
app = "vxapo"
schema = 1

[[effects]]
type = "peq"
name = "footstep boost"          # optional APP metadata, driver ignores
group = "FPS preset"             # optional APP metadata, driver ignores
crossover_hz = 200
[[effects.bands]]
fc = 1000
gain_db = 3.0
q = 1.0
```

Effect types in v1: `peq`, `preamp`, `aural`, `reverb`, `maximizer`, `wide`, `loudness`. `enabled = false` bypasses; `channels = ["FL", "FR"]` restricts scope (default all).

Validation: unknown type / unknown key / inapplicable field / missing required key / out-of-range / PEQ band count not in [1, 31] (global total <= 31) / duplicate or non-device channel name -> whole-file parse failure; old chain is retained.

Fingerprint: `EffectConfig::spec()` (stable serialization of DSP fields; `name`/`group`/`meta` not included). The watcher monitors `config.toml`.

## 6.0 `config/error.rs`

```rust
#[derive(Debug, Clone)]
pub enum ConfigError {
    IoError { path: String, message: String },
    SyntaxError { file: String, line: usize, message: String },
    TomlError { file: String, message: String },
    ModelError { file: String, message: String },
    AbortFile,
}
```

## 6.1 `config/parser.rs`

Parses TOML to `FileModel`, converts to `ChainModel`, and calls `factory::create_from_model` to construct the Filter chain. Produces a config fingerprint (`SpecChain`) for hot-reload comparison.

```rust
pub type FilterSpec = String;
pub type SpecChain = Vec<FilterSpec>;

pub struct ConfigParser;

impl ConfigParser {
    pub fn new() -> Self;
    pub fn parse_file(&self, path: &str, ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;
    pub fn parse_file_with_spec(&self, path: &str, ctx: &DspContext)
        -> Result<(Vec<Box<dyn Filter>>, SpecChain), ConfigError>;
    pub fn parse_string(&self, content: &str, ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;
    pub fn parse_lines(&self, lines: &[String], ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;
}
```

Missing file = passthrough (v9.12). Parse/model validation failure = whole-file failure; callers decide degradation (Lock -> passthrough, hot reload -> keep old chain).

## 6.2 `config/watcher.rs`

Watches the per-device `config.toml`. On change, triggers object-layer hot reload. It does not know install/object details beyond the callback contract.

## 6.3 Historical `config/commands/*`

The old `config/commands/*` line-command system is removed. These sections are retained only as historical notes; the current implementation uses the TOML model and `pipeline/dsp/factory.rs` static dispatch.
