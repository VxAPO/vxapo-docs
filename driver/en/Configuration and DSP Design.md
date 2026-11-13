# VxAPO Driver Configuration and DSP Design

## 1. config.toml model

Each device has one config file:

```text
C:\ProgramData\VxAPO\{GUID}\config.toml
```

The config is TOML v1. Top-level fields:

```toml
version = 1
enabled = true             # false = whole-chain passthrough, content kept

[meta]
app = "vxapo"
schema = 1

[[effects]]
type = "peq"
enabled = true
crossover_hz = 200
name = "example"           # APP metadata, ignored by driver
group = "example group"    # APP metadata, ignored by driver
[[effects.bands]]
fc = 1000
gain_db = 3.0
q = 1.0
```

### 1.1 Model layers

- `FileModel` (`config/model.rs`): TOML file format including APP metadata.
- `ChainModel` (`pipeline/dsp/model.rs`): DSP semantic model without APP metadata.
- Conversion happens in `config`; dependency is `config -> pipeline/dsp`.

### 1.2 Validation

- Unknown `type`, unknown keys, missing required keys, out-of-range values, or type errors fail the whole file.
- On failure, the old chain is kept.
- PEQ bands: 1-31 per block; the 31-band cap is per channel for blocks with explicit `channels`, and one shared budget for unscoped blocks (the cross-channel total may exceed 31).
  Blocks with explicit `channels` count against those channels; unscoped blocks count against the shared budget.
- PEQ per-band ranges: `fc 20..20000`, `gain_db -30..+30`, `q 0.1..12`,
  `crossover_hz 20..20000` (default 200).
- The general `[-120, +48]` dB gain range only applies to `preamp.gain_db`; each effect validates its own ranges (see 1.3).
- Legacy mapping: `type = "maximizer"` / `"leveler"` parse as `compressor`; their legacy parameter keys
  (`gain_boost_db` / `max_output_db` / `target` / `lookahead_ms` / `dither` / `target_rms_db` /
  `response_s` / `max_gain_db` / `dynamic_preserve` / `noise_gate_db` / `peak_limit_db`) are accepted
  but ignored, and `CompressorParams` defaults apply.
- Legacy `wide` keys `intensity` / `depth` are no longer accepted: the `air` fallback was removed, and either key on a `wide` block now rejects the config (only explicit `air` is honored).
- `enabled = false` bypasses an effect.

### 1.3 Effect types

| type | parameters (range) | defaults |
|------|--------------------|----------|
| `peq` | `crossover_hz` (20..20000), `bands` (fc/gain_db/q/type) | `200` / required |
| `preamp` | `gain_db` (-120..48) | required |
| `aural` | `tune_hz` (500..10000), `drive` (0..4.25), `odd` (0..1.5), `even` (0..0.75), `wet`/`dry` (0..1) | `1760 / 1.76993 / 1 / 0.2 / 0.5 / 0.5` (odd/even were 1.5/0.25; odd sat on the range ceiling and over-generated harmonics) |
| `reverb` | `room_size` (0.5..1.5), `decay`/`damping`/`bandwidth`/`density`/`lat5`/`lat6` (0..1), `pre_delay_ms` (0..100), `motion_rate` (0.05..2), `motion_depth` (0..2, legacy `motion_depth_ms`), `low_cut_hz` (20..250), `wet`/`dry` (0..1) | `1.0 / 0.41 / 0.408290 / 0.350110 / 1.0 / 0.70 / 0.50 / 0 / 0.110871 / 0.63 / 100 / 0.27 / 0.73` |
| `compressor` | `threshold_db` (-48..0), `ratio` (1..20), `lift` (0..1), `attack_ms` (1..100), `mix` (0..1) | `-12 / 3 / 0 / 10 / 1` |
| `wide` | `gain`/`air`/`side_itd` (0..1), `crossover_hz` (200..1000) | `0.05 / 0.2 / 0.2 / 200` |
| `loudness` | `phon` (0..120, required), `reference_phon` (0..120) | `reference_phon 80` |

PEQ band types (`bands[].type`, default `peaking`):

| type | meaning |
|------|---------|
| `peaking` | RBJ peaking EQ |
| `low_shelf` / `high_shelf` | shelf filters; `gain_db` is the target level |
| `low_pass` / `high_pass` | RBJ pass filters have no gain term, so `gain_db` is applied linearly to the **passband**; `0 dB` keeps pure filter behavior |

Implementation notes:

- `reverb`: Dattorro plate reverb; `low_cut_hz` is the low-frequency transient protection crossover
  (content below it bypasses the reverb per channel and passes through untouched, `20` ~ off;
  content above the crossover enters the reverb).
- `compressor`: all-channel linked **two-stage detection**. The primary detector is a
  **slow loudness axis** (asymmetric mean-square pole: 0.5 s rise keeps the
  note-scale translation, 0.2 s fall keeps post-passage recovery snappy — symmetric
  0.5 s needed 1.5 s+ to recover from deep reduction, the source of the "sluggish
  muffle"; plus 10 dB crest compensation that aligns with the former peak scale so
  the threshold knob's trigger point stays roughly put; material whose crest
  deviates from the typical value shifts it by a few
  dB): gain stays nearly constant at the note/transient scale, so peaks travel with
  their bodies as a **uniform translation** and the crest is preserved — the old
  peak-envelope drive let attack leak the spike while the body took the full cut
  (per-hit spike/body offset gap up to 9 dB, crest inflated), the source of the
  "forced down, unnatural" feel. The **peak envelope** (instant attack plus
  depth-weighted release, 30 dB/s deep / 180 dB/s shallow) is demoted to **lift
  capping only**: `lift = u × min(max(threshold − peak envelope, 0), 24 dB)` never
  crosses the threshold, and a sudden loud transient drops the boost instantly (no
  boosting of momentary large signals). The curve is a fixed 3 dB soft-knee static
  curve (**branching decided by the axis reading**: `ratio` cuts above the threshold,
  `lift` raises below it; C1-continuous Hermite knee that degenerates bit-exactly to
  the old interpolation at `lift = 0`) + dB-domain attack/release smoothing
  (**release is fixed automatic, no parameter**: dual-pole 100 ms / 600 ms whose
  slow-pole weight scales with the **cube** of reduction depth — deep reductions keep
  the 0.3/0.7 blend and its 600 ms tail (sustained material and bass do not pump),
  shallow ones lean fully on the fast pole) +
  **bounded output**
  (no makeup: `|out| ≤ |in|` always holds at `lift = 0`, and with `lift > 0` the boost
  is capped by the peak envelope so the output level never exceeds the threshold — no
  clipping either way; `mix` blends with the dry signal, `dry = 1 - mix`); replaces
  the former `maximizer` / `leveler`, legacy `knee_db` / `makeup_gain_db` / `wet` /
  `dry` keys are silently ignored;
  `release_ms` was removed together with manual release (now an unknown key — its
  presence rejects the config).
- `wide`: linear-phase FIR crossover; `gain` (HF compensation) only sets the depth of the
  **low-end reduction** — an RBJ low shelf with Q fixed at 0.707, corner = `crossover_hz`, depth =
  6 dB x `gain`, applied to the **reconstructed dry sum**; `air` is the center air absorption;
  `side_itd` (Side ITD) sets the **fractional delay length** of the side channel above 1.5 kHz
  (pure delay, amplitude untouched: 0 = phase untouched, 1 = 0.10 / 0.15 ms, dtau = 0.05 ms,
  scaled by sample rate so the time amount is sample-rate independent). The side dry branch uses that splitter's low-pass branch
  (same group delay as the high band, avoiding an x[n]-x[n-D2] residual), and the mid references
  plus the dry signal are shifted by the splitter's group delay D2 — total latency is therefore
  D2 (~2.6 ms, 127 samples at 48 kHz) above the main crossover. The side channel is neither boosted nor separately
  attenuated; the processed increment passes a first-order high-pass at the crossover, and the
  output has a soft-knee limiter.
- `aural`: 2nd-order Butterworth high-pass + level follower + tanh odd-order soft saturation +
  half-wave rectified even-order term, wet/dry mixed.

## 2. Fingerprint and hot reload

- Fingerprint is generated by `EffectConfig::spec()` and contains only DSP fields.
- `name`, `group`, `meta` are not included.
- The watcher monitors `C:\ProgramData\VxAPO\{GUID}`; changes trigger `hot_reload`.
- Hot reload parses outside the lock, swaps chain pointers inside the lock, and uses a 10 ms raised-cosine transition.

## 3. Hybrid PEQ architecture

PEQ uses a 200 Hz crossover hybrid architecture:

- `Fc < 200 Hz`: IIR biquad path.
- `Fc >= 200 Hz`: FIR path.
- If a high-frequency band deviates more than 0.25 dB at the crossover, it also goes to IIR for low-frequency accuracy.
- Band types: `peaking` / `low_shelf` / `high_shelf` / `low_pass` / `high_pass` (default peaking).
  For `low_pass` / `high_pass`, `gain_db` is applied linearly to the passband (0 dB is identical to a
  pure filter), so one band can express both cutoff and passband level.

Processing chain:

```text
input -> IIR cascade -> minimum-phase FIR -> output
```

FIR tap count is sample-rate adaptive:

```text
N = next_pow2(round(sr * 0.0213)), clamped to [1024, 8192]
```

- `N <= 2048`: direct time-domain FIR.
- `N > 2048`: block FFT, partition size 128.
- Latency is not reported: `latency_frames_atomic` is always 0.

## 4. DSP numerical safety

### 4.1 Background

Extreme parameters (e.g. ±1000 dB) can cause:

- `db_to_linear` overflow to `inf`.
- biquad poles near the unit circle, causing oscillation.
- gain jumps causing pops.

### 4.2 Three defense layers

1. Parse layer: clamp parameters to safe ranges.
2. Coefficient layer: compute biquad in f64 and validate finite/stability.
3. RT layer: set FTZ/DAZ to flush denormals and guard NaN/Inf.

### 4.3 Key constants

| Constant | Value | Description |
|----------|-------|-------------|
| `GAIN_DB_MIN` | -120.0 | minimum gain |
| `GAIN_DB_MAX` | 48.0 | maximum gain |
| `FILTER_CUT_FLOOR_DB` | -60.0 | deep-cut floor |
| `Q_MIN` / `Q_MAX` | 0.05 / 18.0 | safe Q range |
| `MAX_GAIN_STEP_RATIO` | 0.05 | gain smoothing step |
| `MAX_PEQ_BANDS` | 31 | PEQ band limit |
| PEQ `crossover_hz` | 20.0 / 20_000.0 | PEQ crossover range (default 200) |
| PEQ `fc` | 20.0 / 20_000.0 | per-band frequency range |
| PEQ `gain_db` | -30.0 / 30.0 | per-band gain range |
| PEQ `q` | 0.1 / 12.0 | per-band Q range |

## 5. EqualizerAPO behavior reference

This section is the comparison baseline against EAPO for install / child APO / format negotiation behavior (the original Equalizer behavior document was retired in v9.11 and removed after v9.15 in the documentation reorganization; this section is the only surviving reference). The current implementation no longer supports EAPO `config.txt` syntax, but retains comparable mechanisms for:

- Child APO creation and delegation.
- Install mode detection (LfxGfx / SfxMfx / SfxEfx).
- Slot backup and restore.
- Post-install self-check (`CoCreateInstance`).

This document defers to the current code for detailed source locations.

The condition list below was verified line-by-line against the EAPO source (read of `D:\Source_Code\equalizerapo-code` on 2026-08-03: EqualizerAPO.cpp / DeviceAPOInfo.cpp / FilterEngine.cpp/.h / FilterConfiguration.cpp / parser/RegistryFunctions.cpp / IFilter.h); confirmed behavior carries a source location and is kept distinct from inference.

### 5.1 Child APO lifecycle

| # | Behavior (source-verified) | Source location | VxAPO alignment |
|---|----------------------------|-----------------|-----------------|
| C1 | **Initialize reads the child APO GUID**: `APOInitSystemEffects.APOInit.clsid` selects PreMix/PostMix (122); `DeviceAPOInfo` reads `PreMixChildGuid`/`PostMixChildGuid` (160-163) | EqualizerAPO.cpp 122/149-163 | **Fully aligned**: object 7.1.8 — the child APO source is the previous occupant of the takeover slot |
| C2 | **Creation sequence**: `CLSIDFromString` → `CoCreateInstance(childGuid, NULL, CLSCTX_INPROC_SERVER, IID_IAudioProcessingObject)` → QI `IAudioProcessingObjectRT` → QI `IAudioProcessingObjectConfiguration` | EqualizerAPO.cpp 178-209 | **Fully aligned**: the three-step create in `object/child.rs` |
| C3 | **Initialize passthrough**: `childAPO->Initialize(cbDataSize, pbyData)` forwards the **same parent APOInit data** | EqualizerAPO.cpp 211 | **Fully aligned**: object 7.1.8 |
| C4 | **Creation-failure degradation**: any failing step (CoCreate / QI RT / QI Cfg / Initialize) → `resetChild()` → **returns S_OK** (does not block the parent) | EqualizerAPO.cpp 186-217 | **Fully aligned**: `child_apo = None` degradation (object 7.1.8 Note 57) |
| C5 | **GetLatency**: `*pTime = 0` → **delegates only when a child exists** via `childAPO->GetLatency(pTime)` → returns S_OK; **with no child it returns 0** (not "falls back to its own latency") | EqualizerAPO.cpp 82-95 | **Aligned (corrected)**: object 7.2 GetLatency delegation; returns 0 with no child |
| C6 | **IsInputFormatSupported delegation**: with a child → `childAPO->IsInputFormatSupported` (260); **on child failure → `resetChild()` destroys the child** (268) → falls back to the self check (272-283) | EqualizerAPO.cpp 258-283 | **Aligned + supplement**: child failure **destroys** the child (not merely a fallback); VxAPO aligns the resetChild semantics |
| C7 | **LockForProcess delegation**: with a child config → `childCfg->LockForProcess` (the result is **only traced, not returned**, 343-346) → the parent then locks itself | EqualizerAPO.cpp 341-350 | **Aligned**: object 7.1.9 — a child refusing the lock degrades instead of failing |
| C8 | **UnlockForProcess failure semantics (EAPO side)**: `childCfg->UnlockForProcess` FAILED → **return hr** (the parent unlock fails) | EqualizerAPO.cpp 393-401 | **VxAPO fault-tolerance difference** (object 7.1.10): a child unlock failure does not block the parent |
| C9 | **resetChild semantics**: the three interfaces (childAPO / childRT / childCfg) are each `Release()`d and set to NULL | EqualizerAPO.cpp 406-424 | **Aligned**: ChildApo Drop (object 7.2) |

### 5.2 APOProcess delegation and timing

| # | Behavior (source-verified) | Source location | VxAPO alignment |
|---|----------------------------|-----------------|-----------------|
| C10 | **APOProcess delegation order**: **`childRT->APOProcess` runs first** (on the input buffer) → `engine.process(outputFrames, outputFrames, ...)` (**in-place processing of the output buffer**) | EqualizerAPO.cpp 472-477 | **Fully aligned**: object 7.1.11 — the child runs first, once per frame |
| C11 | **BUFFER_SILENT input zeroing**: when the input is SILENT the input buffer is `memset` first (468-470) → childRT processes → silent-output decision (with allowSilentBufferModification 486-499; otherwise zero the output + BUFFER_SILENT 503-505) | EqualizerAPO.cpp 468-511 | **Aligned + v9.15 tightening**: VxAPO does not read stale data on SILENT input, treats it as all-zero, and forces SILENT output (pipeline 4.2 / object 7.1.11) — fixes "engine reuses a dirty silent buffer → self-feedback pop" |
| C12 | **Child output channel semantics**: `realChannelCount` = outFormat channel count **with a child**; inFormat channel count **without a child** | EqualizerAPO.cpp 365-369 | **Equivalent stance**: VxAPO does not introduce a realChannelCount mechanism (main spec 18.2 D2) |
| C13 | **realChannelCount includes child output**: field comment "input channel count during processing (including child APO output)" | FilterEngine.h 131-132 | **Equivalent**: main spec 18.2 D2/D3 |

### 5.3 Slot installation and backup

| # | Behavior (source-verified) | Source location | VxAPO alignment |
|---|----------------------------|-----------------|-----------------|
| C14 | **installPostMix defaults to `!input`** (only render devices get PostMix) | DeviceAPOInfo.cpp 254 | **Aligned**: object 7.2 |
| C15 | **useOriginal defaults**: `useOriginalAPOPreMix = true`, `useOriginalAPOPostMix = !input` | DeviceAPOInfo.cpp 255-256 | **Aligned**: object 7.2 predecessor-APO backup |
| C16 | **All slots backed up**: current values of every FxProperties slot (allGuidValueNames) → written to `childApoPath\{deviceGuid}` (567-577, same area) — **including NOKEY/NOVALUE placeholders** | DeviceAPOInfo.cpp 529-548 | **Aligned**: object 7.2 backs up all slots for rollback |
| C17 | **Predecessor APO backup `.reg` file**: when FxProperties exists, `saveToFile` writes `backup_*.reg` (551-553) | DeviceAPOInfo.cpp 550-553 | **VxAPO difference**: VxAPO keeps no `.reg` file backup (object 7.2 registry backup only) — deferred as P1, on demand |
| C18 | **autoAdjust write**: `autoAdjust=true` → deletes the `disableAutoAdjust` key; **false → writes `disableAutoAdjust="true"`** (the default) | DeviceAPOInfo.cpp 567-575 | **Aligned + conservative default**: VxAPO defaults to false (install 5.5.2 E3.3) |
| C19 | **allowSilentBuffer write**: writes `allowSilentBufferValueName` (true/false) | DeviceAPOInfo.cpp 565-566 | **Aligned**: object 7.1.2 allow_silent_buffer_modification |

### 5.4 Format negotiation and channel semantics

| # | Behavior (source-verified) | Source location | VxAPO alignment |
|---|----------------------------|-----------------|-----------------|
| C20 | **Downmix rejection**: `inFormat > 2ch && inFormat.dwSamplesPerFrame > outFormat.dwSamplesPerFrame` → builds an outFormat media type + `S_FALSE` (**the parent does not downmix**) | EqualizerAPO.cpp 277-282 | **Equivalent stance (corrected wording)**: during negotiation VxAPO pins "received input == output channel count" (the parent converts nothing); "reject in>out" is precisely the downmix rejection, not a blanket "pin in==out" |
| C21 | **mono→stereo upmix special case**: `realChannelCount==1 && outputChannelCount>=2` → copies mono to channel 1 (comment: "simulates the automatic upmix the Windows audio system does when no APO is present") | FilterConfiguration.cpp 126-128 | **Equivalent stance**: only this upmix special case exists; the parent performs no other up/downmix (main spec 18.2) |
| C22 | **Negotiation delegation + destroy-on-child-failure fallback**: child success → the child decides; child failure → **resetChild destroys it** + fall back to the self check | EqualizerAPO.cpp 258-283 | **Aligned + narrowed**: the VxAPO three-stage whitelist (main spec 18.2 D5) |

### 5.5 Configuration parsing behavior

| # | Behavior (source-verified) | Source location | VxAPO alignment |
|---|----------------------------|-----------------|-----------------|
| C23 | **Lines without a colon are skipped**: `pos = line.find(':')`; **`pos == -1` → the whole line is skipped without parsing** (no error, no rejection) | FilterEngine.cpp 329-330 | **Aligned (corrected)**: the VxAPO v7.11 "reject colon-less lines" is **stricter than EAPO** — EAPO silently skips; VxAPO reports an explicit error (an intentional difference under the "syntax strictness" intent) |
| C24 | **Read failure returns**: `CreateFile` failure (other than a sharing violation) → `return` (keeps the current config, no replacement) | FilterEngine.cpp 285-287 | **Aligned**: object 7.1.18 — a parse failure keeps the old chain |
| C25 | **Config-file wait semantics**: `ERROR_SHARING_VIOLATION` while the file is being written → `Sleep(1)` and retry (not an error) | FilterEngine.cpp 282-292 | **VxAPO difference**: VxAPO uses a 128 KB gate + spec comparison (config 6.1); EAPO has no size gate |

### 5.6 Configuration load and transition (FilterEngine / FilterConfiguration)

| # | Behavior (source-verified) | Source location | VxAPO alignment |
|---|----------------------------|-----------------|-----------------|
| C26 | **previousConfig deferred destruction**: `loadConfig` **begins** by destroying previousConfig (on the control thread) | FilterEngine.cpp 219-224 | **Aligned**: R1 retired-chain deferred destruction (object 7.1.3/7.1.11) |
| C27 | **loadSemaphore blocking reload**: a 1/1 semaphore is created (79); `ReleaseSemaphore` fires when the transition completes (413/452) → the notification thread waits on the semaphore before loadConfig (606-614) | FilterEngine.cpp 79/413/452/606-614 | **Aligned**: R2 blocking reload (object 7.1.18) |
| C28 | **transitionLength = sampleRate/100 (10 ms)** | FilterEngine.cpp 154 | **Aligned**: R4 (pipeline 4.11) |
| C29 | **The EAPO transition is config-level**: old and new allSamples planes are blended per frame by `doTransition` (raised cosine); **the next config is released once `transitionCounter >= transitionLength`** (407-414) | FilterConfiguration.cpp 157-176 + FilterEngine.cpp 402/407-414 | **VxAPO difference (D1)**: in-parent dual-chain transition (outgoing/current mix_buffers) |
| C30 | **Buffer preallocation**: the FilterConfiguration constructor preallocates allSamples/allSamples2 by `maxFrameCount` (zero allocation on the RT path) | FilterConfiguration.cpp 31-52 | **Aligned**: object 7.1.9 LockForProcess preallocates temp_buffer_old/new |
| C31 | **Notification-thread creation timing**: at the end of `engine.initialize`, when `configPath != ""` (**initialize is called on the LockForProcess path**) | FilterEngine.cpp 201-205 | **Aligned**: object 7.1.9 — the watcher lifetime follows the lock cycle |
| C32 | **Directory change notification**: `FindFirstChangeNotificationW(configPath, recursive, file name | last write)` + **10 ms deduplication** (599-604) + wait for loadSemaphore, then loadConfig (606-614) | FilterEngine.cpp 556-624 | **Aligned**: config 6.2 directory-level event-driven + 10 ms dedup |
| C33 | **watchRegistryKey is a readReg side effect**: after a successful `readRegString` (52) / `readRegDWORD` (92) read → `engine->watchRegistryKey(key)` → the notification thread watches via `RegNotifyChangeKeyValue` (577) | parser/RegistryFunctions.cpp 52/92 + FilterEngine.cpp 375-378/577 | **Not needed in VxAPO**: config is pure files, there is no readReg command (object 7.2) |
| C34 | **isEmpty empty-chain fast path**: `currentConfig->isEmpty() && nextConfig==NULL` → straight memcpy (avoids the deinterleave cost) | FilterEngine.cpp 383-393/419-431 | **Aligned**: R3 `Chain::is_empty()` fast path (pipeline 4.5/4.6) |
| C35 | **getInPlace in-place declaration**: `IFilter::getInPlace()` (default true); addFilters reads `filterInfo->inPlace` (464) | IFilter.h 41 + FilterEngine.cpp 464 | **Aligned**: E1 `Filter::is_in_place()` (pipeline 4.9) |

### 5.7 Installation and devices (E3.x)

| # | Behavior (source-verified) | Source location | VxAPO alignment |
|---|----------------------------|-----------------|-----------------|
| C36 | **Default-device decision boundary (E3.1)**: the driver-layer enumeration (`loadAllInfos`) only returns installed containers; the default device is obtained via `GetDefaultAudioEndpoint` (**user layer / API layer**) and matched against the driver results | DeviceAPOInfo.cpp 82-87/106-116 | **Aligned**: install 5.4 E3.1 |
| C37 | **testAPOInstallation (E3.4 corrected)**: activates `IAudioClient` (Activate + GetMixFormat + Initialize, shared mode, 100 ms) — a **full audio-pipeline self-check**, **not** "CoCreateInstance verifying the DLL can be instantiated"; on failure `fail()` throws `DeviceException` (the exception propagates) | DeviceAPOInfo.cpp 777-815 | **VxAPO must revise the E3.4 description**: the install 5.5.2 "CoCreateInstance verifies the DLL" description **does not match the EAPO source** — EAPO actually activates IAudioClient for a pipeline self-check |

### 5.8 Installation privilege mechanism (EAPO install session)

> **Conclusion: EAPO uses "compile-time manifest declarative elevation" throughout; there is no "runtime detection → elevate → restart" code anywhere.**

| # | Behavior (source-verified) | Source location | VxAPO lesson |
|---|----------------------------|-----------------|--------------|
| C38 | **The NSIS installer elevates as a whole**: `RequestExecutionLevel admin` — UAC triggers at installer startup and the whole flow runs under an admin token (regsvr32 + DeviceSelector.exe /i silent install + HKLM writes) | Setup.nsi 35 | The VxAPO CLI installer can align (the installer needs admin) |
| C39 | **Configurator/DeviceSelector manifest `RequireAdministrator`**: **all 6 build configurations** set `<UACExecutionLevel>RequireAdministrator</UACExecutionLevel>` — **every launch** requires admin, so writing FxProperties needs no runtime elevation | DeviceSelector.vcxproj 169/184/200/216/233/248 | **Key lesson for the VxAPO CLI**: writing HKLM (install/uninstall) needs admin; declare it via manifest or elevate explicitly |
| C40 | **Logon scheduled task (non-elevated)**: `scheduleAtLogon` registers a "logon trigger + network available" task running **UpdateChecker** (`TASK_LOGON_INTERACTIVE_TOKEN`); `singleInstance` guards the update check — unrelated to elevation | TaskSchedulerHelper.cpp 34-186 | VxAPO can reuse the scheduled-task pattern if it adds an update check |

**Supplement (corroborating C39)**: the program contains no `ShellExecuteEx(runas)` / `IsUserAnAdmin()` / `CreateProcess` elevation calls (confirmed by source search); **everything relies on the manifest**. Outside the program: installer `RequestExecutionLevel admin` (Setup.nsi 35) + Configurator `RequireAdministrator` (DeviceSelector.vcxproj, 6 places).

### 5.9 Installation slot mechanism (mode probing + mutually exclusive writes)

> **Behavior flow**: `load()` probes the install mode (the Win8.1+ three tiers) → `install()` writes the target slot exclusively for the selected mode + deletes out-of-mode slots + per-device backup.
> **Session boundary**: DeviceSelector.exe triggers UAC at startup → loads `loadAllInfos` → the user runs `install()` per device (admin token, no runtime elevation).

| # | Behavior (source-verified) | Source location | VxAPO alignment |
|---|----------------------------|-----------------|-----------------|
| C41 | **LfxGfx mode decision**: Win8.1+ and FxProperties has **only LFX/GFX values and no SFX/MFX/EFX values** → the driver supports Legacy only | DeviceAPOInfo.cpp 396-408 | **Aligned**: install 5.1 mode detection (Legacy exclusive) |
| C42 | **SfxMfx mode decision**: `combinedDeviceValueName` detected (**Win11 Bluetooth combined device, EFX invalid**) — the value name is exactly `{b3f8fa53-0004-438e-9003-51a46e139bfc},41` (**PKEY_Device_ContainerId, the device container ID**, WT_DEVICE property set PID 41), located in the endpoint `Properties` subkey (`keyPath\Properties`, not FxProperties) | DeviceAPOInfo.cpp 51/410-411 | **Deliberate divergence**: EAPO detects Bluetooth combined devices via the container-ID value name `{b3f8fa53-...},41` (PKEY_Device_ContainerId — present on all devices, so USB devices get misjudged as SfxMfx); VxAPO instead detects Bluetooth via the endpoint instance-ID prefixes BTHENUM/BTHLE (`detect_install_mode` in install/device/slots/read.rs, info.rs:251-285) |
| C43 | **SfxEfx default mode**: otherwise (the modern-driver default) | DeviceAPOInfo.cpp 412-413 | **Aligned**: SfxEfx is the default (install 5.1) |
| C44 | **Old Windows (<8.1) defaults to LfxGfx**: `installMode` starts as `INSTALL_LFX_GFX` (249); probing happens only on 8.1+ | DeviceAPOInfo.cpp 249/396 | **Aligned**: install 5.1 mode default |
| C45 | **Three-mode mutually exclusive write**: `install()` writes the target slot per mode + **deletes out-of-mode slots** (LfxGfx deletes SFX/MFX/EFX; SfxMfx deletes LFX/GFX and keeps EFX; SfxEfx deletes LFX/GFX and keeps MFX) | DeviceAPOInfo.cpp 578-640 | **Aligned**: install 5.5.2 writes slots per mode |
| C46 | **Processing-mode values written per slot**: the newly installed slot writes `{sfx/mfx/efx}ProcessingModesValueName` (not overwritten when already present), value = `defaultProcessingModeValue` | DeviceAPOInfo.cpp 603-605/611-613/627-629 | **Aligned**: install 5.5.2 Step 6 (default processing modes) |
| C47 | **FxProperties backup precedes writes**: when present → each value (NOKEY/NOVALUE placeholder or the actual value) is written to childApoPath + a `.reg` backup; when absent → create it + NOKEY placeholders for all 5 slots | DeviceAPOInfo.cpp 512-554 | **Aligned**: object 7.2 backs up all slots |
| C48 | **Capture devices install PreMix only**: `installPostMix && !input` (PostMix is for render devices only) | DeviceAPOInfo.cpp 583/607/632 | **Aligned**: object 7.2 installPostMix=!input |
| C49 | **Installed devices keep their mode**: `load()` detects `foundAt` (LFX/SFX → premix; GFX/MFX/EFX → postmix) → keeps the existing install state | DeviceAPOInfo.cpp 339-352 | **Aligned**: install 5.1 keeps the installed state |
| C50 | **Version upgrade**: `canBeUpgraded()` = `installed && version != installVersion` ("2") | DeviceAPOInfo.cpp 421-423 | **Aligned**: install 5.4 can_be_upgraded |

### 5.10 GraphicEQ and CAPX "device default effects" (source-verified 2026-08-10)

| # | Behavior (source-verified) | Source location | VxAPO alignment |
|---|----------------------------|-----------------|-----------------|
| C51 | **The EAPO GraphicEQ is not a biquad cascade**: `GraphicEQFilter` extends `ConvolutionFilter`; in `initializeFilters` the node gains are interpolated over frequency into a frequency response, an FFT-based minimum-phase transform produces an FIR, and the filter convolves with it | GraphicEQFilter.cpp 44-101 | **VxAPO v9.0 + v9.7 aligned**: `pipeline/dsp/graphic_eq.rs` log-frequency interpolation + minimum-phase FIR + **1024-point direct time-domain convolution (AVX2/FMA vectorized)** (the v9.5 blocked-FFT block-buffered stream stopped losing note tails but buzzed on track switch; v9.6/v9.7 finalized the direct FIR) |
| C52 | **Log-frequency linear interpolation between nodes**: `GainIterator::gainAt` interpolates linearly on `log(freq)`; below the first / above the last node the endpoint gains hold (flat out of band) | GainIterator.cpp 30-98 | **Aligned**: `gain_at()` with the same semantics |
| C53 | **GetLatency returns 0 with no child**: even though convolution is used internally (inherent filter delay), nothing is reported to the engine; VxAPO uses a 1024-point direct FIR (v9.7, no block buffering) and keeps aligning with this behavior | EqualizerAPO.cpp 82-95 | **Aligned**: object 7.2 / the 2026-08-10 latency policy |
| C54 | **EAPO does not process the CAPX `MSFX\N` template**: generic USB devices get "Microsoft Audio Home Theater Effects" (the two WMALFXGFX APOs) registered on the device interface by `wdma_usb.inf`; a Windows restart / endpoint re-enumeration may restore the Microsoft APOs from the template, and EAPO does not take them over | wdma_usb.inf `USBAudio.SysFx.Render` | **VxAPO v9.0 + v9.4 extension**: install `device/sysfx.rs` locates and replaces the StreamEffect/ModeEffect of `MSFX\N` (restored on uninstall); from v9.4 the DLL `Initialize` also **self-heals at runtime** — Microsoft CAPX values re-injected by Windows after re-enumeration are taken over again by this DLL (touches only Microsoft CLSIDs, idempotent, degrades gracefully on failure) |
| C55 | **Forced enhancement enable**: EAPO install deletes `{1da5d803-d492-4edd-8c23-e0c0ffee7f0e},5` (PKEY_AudioEndpoint_Disable_SysFx); `fxTitle` is written only when FxProperties is created anew | DeviceAPOInfo.cpp 642-645 / 527 | **Aligned + extension**: VxAPO deletes that value too; fxTitle is not written (avoids historical volume/format problems) |
| C56 | **From v9.11 VxAPO no longer aligns with the EAPO EQ system**: the `GraphicEQ:` command and `graphic_eq.rs` were removed, replaced by the TOML `[[effects]] type="peq"` hybrid PEQ (200 Hz crossover: Fc<200 bands in an IIR cascade, Fc≥200 bands in a sample-rate-adaptive minimum-phase FIR, 1024–8192 taps); command parsing / factory registration no longer align with EAPO; latency policy (finalized v9.12) — `GetLatency` always 0 and `latency_frames_atomic` always 0 (hidden latency is not reported; v9.11 briefly tried engine-frame compensation on activation, but live testing stalled hot reload / track switching, so it was reverted) | — (independent design) | **VxAPO v9.11 + v9.12**: the config TOML design document + the PEQ design document + pipeline 4.10/4.18/4.22 + config 6.0 |
| C57 | **Hot-reload monitoring has no cross-instance deduplication (v9.15 regression alignment)**: per-instance notification thread + 10 ms dedup + loadSemaphore blocking reload (FilterEngine.cpp 556-624); **no process-wide global lock and no mtime dedup table** | FilterEngine.cpp 556-624 | **VxAPO v9.15 aligned**: removed the v9.4 (mtime, size) pre-check and the experimental global lock, restoring "per-instance independent reload + spec fingerprint short-circuit + transition/pending" (object 7.1.9/7.1.18); keeps the diag write lock to prevent garbled log output |

> For more detailed `config/` and `pipeline/` module specifications, see `driver/en/Module Reference Specification/`.
