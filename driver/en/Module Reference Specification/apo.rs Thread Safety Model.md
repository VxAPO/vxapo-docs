# Thread Safety Model (`object/apo.rs`)

## Calling threads

| Method | Thread | Trigger |
|--------|--------|---------|
| `CreateInstance` | application thread | COM object creation |
| `Initialize` | application thread | APO initialization |
| `LockForProcess` | application thread | audio format negotiation |
| `UnlockForProcess` | application thread | audio stream stop |
| `GetRegistrationProperties` | application thread | registration property query |
| `IsInputFormatSupported` | application thread | format negotiation |
| `IsOutputFormatSupported` | application thread | format negotiation |
| `GetInputChannelCount` | application thread | channel count query |
| `CalcInputFrames` | Windows audio engine real-time thread | frame count calculation |
| `CalcOutputFrames` | Windows audio engine real-time thread | frame count calculation |
| `APOProcess` | Windows audio engine real-time thread | per-frame audio processing |
| `GetLatency` | application thread | latency query |
| `Reset` | application thread | state reset |
| `hot_reload` | config watcher background thread | config.toml change |

## Protection mechanisms

| Protected state | Mechanism | Access paths |
|-----------------|-----------|--------------|
| `ApoObjectInner` (chain, transition, pipeline_context, temp_buffers, pending_reload) | `self.mutex: Mutex<ApoObjectInner>` | APOProcess, LockForProcess, UnlockForProcess, Reset, hot_reload |
| `ApoObjectState` (clsid, is_locked, sample_rate, channels, bits_per_sample) | `self.ap_state: Mutex<ApoObjectState>` | LockForProcess, UnlockForProcess, GetRegistrationProperties, GetInputChannelCount |
| State machine (Created / Initialized / Locked) | `self.state_cell: StateCell` (`AtomicU8` + CAS) | Initialize, LockForProcess, UnlockForProcess, APOProcess (read-only check) |
| Latency samples | `self.latency_samples: AtomicU32` | LockForProcess (write), GetLatency (read), Reset (write) |
| Latency frames | `self.latency_frames_atomic: AtomicU32` | always 0 since v9.12; Lock/Reset write 0; CalcInputFrames/CalcOutputFrames read |

## Mutex guarantees

- All fields of `ApoObjectInner` are protected by the same `Mutex`.
- Each method takes the lock, performs its complete operation, then releases.
- `APOProcess` holds the lock for about 1-3 ms depending on filter chain complexity and frame size.
- `LockForProcess` holds the lock for about 10-100 ms (config parsing + Chain construction).
- `hot_reload` parses config and builds the new Chain outside the lock (10-100 ms); inside the lock it only performs `Box` pointer replacement (sub-microsecond).
- `Reset` holds the lock very briefly.
- `APOProcess` and `hot_reload` are mutually exclusive: if `hot_reload` triggers during `APOProcess`, it waits until `APOProcess` releases the lock.

## `self.ap_state` guarantees

- Protects format and channel state.
- Lock hold time is extremely short (field read/write level, sub-microsecond).
- In `LockForProcess` / `UnlockForProcess`, `self.ap_state` is acquired after `self.mutex`, but the two locks are never held at the same time.
- `GetRegistrationProperties` and `GetInputChannelCount` acquire only `self.ap_state`.

## Atomic state cell

- Uses `AtomicU8` + `compare_exchange` (`AcqRel` / `Acquire`) for a lock-free state machine.
- Does not depend on `Mutex`; non-blocking.

## Latency atomics

- `latency_samples` and `latency_frames_atomic` are independently protected by `AtomicU32`.
- Writes happen inside `self.mutex` (LockForProcess / UnlockForProcess / Reset).
- Reads happen outside the lock (CalcInputFrames / CalcOutputFrames read `latency_frames_atomic`, always 0).

## RT blocking acceptability

- `self.mutex` blocking sources are not during playback: `LockForProcess`, `Reset`, and `hot_reload` lock sections are explicit or sub-microsecond.
- `APOProcess` has no competing `APOProcess` because the Windows audio engine calls serially.
- `self.ap_state` is only accessed before playback or during format negotiation.
- Atomics are non-blocking.
- For system APO at 48 kHz / 128 frames (~2.67 ms period), occasional sub-microsecond mutex waits are inaudible.

## Non-reentrancy invariants

- `self.mutex` is not reentrant.
- `self.ap_state` is not reentrant.
- The two locks are never held simultaneously on any execution path.
- `CalcInputFrames` / `CalcOutputFrames` do not acquire any lock; they read `latency_frames_atomic` lock-free.
- `LockForProcess` must not call `CalcInputFrames` / `CalcOutputFrames`; since v9.12 latency is not reported (`latency_frames_atomic` is always 0); v9.13 reuses the existing chain on same config/format Relock (`last_lock_key` fingerprint).
- `hot_reload` must not call methods that require `self.mutex`; the whole `hot_reload` already holds `self.mutex`.
- `GetLatency` reads `pipeline_context.sample_rate` inside `self.mutex` and loads the latency value via `latency_samples`; it does not acquire `self.ap_state`.
- Violating these invariants triggers `std::sync::Mutex` panic in debug builds (lock reentry).
