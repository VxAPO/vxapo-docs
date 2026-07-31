## 九、`telemetry/` 模块规范

### 9.1 `telemetry/logger.rs`

**职责**：无锁环形日志，实时路径零堆分配。

**引用来源**：
- `crate::pipeline::realtime::ring::RingBuffer`

**导出给**：`pipeline/`、`object/`

**公开 API**：
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

    // 实时安全：无堆分配，无阻塞
    pub fn log(&self, level: LogLevel, msg: &str) {
        let bytes = msg.as_bytes();
        let len = bytes.len().min(255);
        let mut entry = LogEntry { level, len: len as u8, msg: [0u8; 256] };
        entry.msg[..len].copy_from_slice(&bytes[..len]);
        let _ = self.ring.push(entry);
    }

    // 非实时：批量读取
    pub fn drain<F>(&self, mut f: F) where F: FnMut(LogLevel, &str) {
        while let Some(entry) = self.ring.pop() {
            let len = entry.len as usize;
            let msg = core::str::from_utf8(&entry.msg[..len]).unwrap_or("<invalid utf8>");
            f(entry.level, msg);
        }
    }
}
```

**消息截断行为**：超过 255 字节的消息被静默截断。RT 日志应保持简短（错误码 + 关键参数）。

好。

### 9.2 `telemetry/panic.rs`

**职责**：panic hook 安装

**引用来源**：
- `crate::telemetry::logger::Logger`

**导出给**：`object/dll_exports.rs`

**公开 API**：
```rust
pub fn install_panic_hook(logger: &'static Logger);
```

**panic hook 行为**：
1. 捕获 panic 信息
2. 写入 logger（RtViolation 级别）
3. `abort()`

**`'static` 生命周期要求**：
`install_panic_hook` 需要 `&'static Logger`，因为 panic hook 是全局的，必须在程序整个生命周期内有效。

在 `object/dll_exports.rs` 中，使用 `std::sync::OnceLock` 创建全局静态 Logger：

```rust
use std::sync::OnceLock;
use crate::telemetry::logger::Logger;

static LOGGER: OnceLock<Logger> = OnceLock::new();

// 在 DllRegisterServer 或 DllMain 中调用
LOGGER.get_or_init(|| Logger::new(1024));
install_panic_hook(LOGGER.get().unwrap());
```

**注意**：不要在 `DllMain` 中执行复杂初始化。推荐在 `DllRegisterServer`（自注册入口）中初始化 Logger。

