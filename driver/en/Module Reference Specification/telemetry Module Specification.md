# telemetry Module Specification

## 9.1 `telemetry/logger.rs`

**Responsibility**: Lock-free ring logging with zero heap allocation on the real-time path.

**References**:
- `crate::pipeline::realtime::ring::RingBuffer`

**Exports to**: `pipeline/`, `object/`

**Public API**:
```rust
#[repr(C)]
#[derive(Copy, Clone)]
pub struct LogEntry {
    level: LogLevel,
    len: u8,
    msg: [u8; 256],
}

#[repr(u8)]
pub enum LogLevel {
    Debug,
    Info,
    Warning,
    Error,
    RtViolation,
}

pub struct Logger {
    ring: RingBuffer<LogEntry>,
}

impl Logger {
    pub fn new(capacity: usize) -> Self;

    // Real-time safe: no heap allocation, no blocking
    pub fn log(&self, level: LogLevel, msg: &str) {
        let bytes = msg.as_bytes();
        let len = bytes.len().min(255);
        let mut entry = LogEntry { level, len: len as u8, msg: [0u8; 256] };
        entry.msg[..len].copy_from_slice(&bytes[..len]);
        let _ = self.ring.push(entry);
    }

    // Non-real-time: batch read
    pub fn drain<F>(&self, mut f: F) where F: FnMut(LogLevel, &str) {
        while let Some(entry) = self.ring.pop() {
            let len = entry.len as usize;
            let msg = core::str::from_utf8(&entry.msg[..len]).unwrap_or("<invalid utf8>");
            f(entry.level, msg);
        }
    }
}
```

**Truncation**: Messages longer than 255 bytes are silently truncated. RT log messages should stay short (error code + key parameters).

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
1. Capture panic information.
2. Write to the logger at `RtViolation` level.
3. Call `abort()`.

**`'static` lifetime requirement**:
`install_panic_hook` requires `&'static Logger` because the panic hook is global and must live for the entire process lifetime.

In `object/dll_exports.rs`, a global static `Logger` is created with `std::sync::OnceLock`:

```rust
use std::sync::OnceLock;
use crate::telemetry::logger::Logger;

static LOGGER: OnceLock<Logger> = OnceLock::new();

LOGGER.get_or_init(|| Logger::new(1024));
install_panic_hook(LOGGER.get().unwrap());
```

**Note**: Do not perform complex initialization in `DllMain`. Prefer `DllRegisterServer` (self-registration entry) for Logger initialization.
