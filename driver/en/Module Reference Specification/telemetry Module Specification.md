# telemetry Module Specification

## 9.1 `telemetry/logger.rs`

**Responsibility**: Lock-free ring logging with zero heap allocation on the real-time path.

**References**:
- `crate::utils::ring::RingBuffer` (SPSC lock-free ring buffer)

**Exports to**: `pipeline/`, `object/`

**Public API**:
```rust
/// Process-level logger (lazily initialized on the first `DllGetClassObject` call).
pub(crate) static LOGGER: OnceLock<Logger> = OnceLock::new();

#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum LogLevel {
    Debug,
    Info,
    Warning,
    Error,
    RtViolation,
}

/// Fixed-size ring log entry (RT-safe).
#[repr(C)]
#[derive(Clone, Copy)]
pub struct LogEntry {
    pub level: LogLevel,
    pub len: u8,
    pub msg: [u8; 256],
}

pub struct Logger {
    ring: RingBuffer<LogEntry>,
}

impl Logger {
    pub fn new(capacity: usize) -> Self;

    /// Lazily initializes the process-level logger (capacity 4096) and installs it
    /// into the `log` crate. The first call happens in `DllGetClassObject`
    /// (outside the Loader Lock), not in `DllMain`. Idempotent: a repeated
    /// `set_logger` failure is ignored (already installed).
    pub fn install() -> &'static Logger;

    // Real-time safe: no heap allocation, no blocking.
    pub fn log(&self, level: LogLevel, msg: &str) {
        let bytes = msg.as_bytes();
        let len = bytes.len().min(255);
        let mut entry = LogEntry { level, len: len as u8, msg: [0u8; 256] };
        entry.msg[..len].copy_from_slice(&bytes[..len]);
        let _ = self.ring.push(entry);
    }

    // Non-real-time batch read. Test-only in the current codebase (`#[cfg(test)]`).
    pub fn drain<F>(&self, mut f: F) where F: FnMut(LogLevel, &str) {
        while let Some(entry) = self.ring.pop() {
            let len = entry.len as usize;
            let msg = core::str::from_utf8(&entry.msg[..len]).unwrap_or("<invalid utf8>");
            f(entry.level, msg);
        }
    }
}

// `log` crate facade: `log::info!`-style macros land in the same ring buffer.
impl log::Log for Logger {
    fn enabled(&self, metadata: &log::Metadata<'_>) -> bool; // gates at `Debug`
    fn log(&self, record: &log::Record<'_>);                 // maps level + `{target}: {args}`
    fn flush(&self) {}                                       // no-op
}
```

**Truncation**: Messages longer than 255 bytes are silently truncated. RT log messages should stay short (error code + key parameters).

**Allocation note**: The `log` macros are not on the RT hot path (RT diagnostics go through the fixed-size RT ring); therefore the facade's `format!` allocation is acceptable there.

## 9.2 `telemetry/panic.rs`

**Responsibility**: Install a panic hook.

**References**:
- `crate::telemetry::logger::Logger`

**Exports to**: `object/dll_exports.rs`

**Public API**:
```rust
pub fn install_panic_hook(logger: &'static Logger);
```

**Panic hook behavior**:
1. Capture the panic payload (`&str`, `String`, or `Debug` fallback) as `panic: ...`.
2. Write it to the logger at `RtViolation` level.
3. Call `std::process::abort()`.

**Idempotency**: Installation is guarded by `std::sync::Once` — `DllGetClassObject` may run once per CLSID (PreMix / PostMix); only the first hook is installed. In test builds the hook is not installed, so intentional `#[should_panic]` tests do not abort the test process.

**`'static` lifetime requirement**:
`install_panic_hook` requires `&'static Logger` because the panic hook is global and must live for the entire process lifetime.

In `object/dll_exports.rs`, telemetry is installed lazily at the first `DllGetClassObject` call (outside the Loader Lock):

```rust
let logger = crate::telemetry::logger::Logger::install();
crate::telemetry::panic::install_panic_hook(logger);
```

**Note**: Do not perform complex initialization in `DllMain`. The process-level `Logger` lives in `telemetry::logger::LOGGER` (`OnceLock`, capacity 4096) and is initialized by `Logger::install()` on the first `DllGetClassObject` call — not in `DllRegisterServer` and not in `DllMain`.
