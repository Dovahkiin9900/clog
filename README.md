# clog: A Lightweight, Thread‑Safe, Colorful C Logging Library

`clog` is a single‑header, thread‑safe logging library for C, offering timestamped, context‑rich output with configurable levels and color modes. It works on POSIX and Windows, and requires no dynamic allocations in the hot path.

## Features

- **Log Levels**: TRACE, DEBUG, INFO, WARN, ERROR, FATAL
- **Thread Safety**:
  Uses Windows Critical Sections, C11 atomics, or GCC/Clang spin-locks; fallback to no-op in single-threaded builds
- **Signal Safety**:
  Provides a minimal, async-signal-safe logging function `clog_signal_log` for use in signal handlers
- **Color Modes**:
  - Auto-detect (ANSI on POSIX TTYs or modern Windows consoles)
  - Force ANSI escapes
  - Force Windows Console API
  - Always/never color
- **Contextual Output**:
  Outputs timestamp, level tag, message, and optional `(file:line in function)` location info
- **Customizable**:
  - `clog_set_level(...)` to filter by minimum level
  - `clog_set_color_mode(...)` to override color behavior
  - `clog_set_output(...)` to redirect logs to any `FILE*`
- **Performance-oriented**:
  Pre-allocated buffers, no `malloc` in log path
- **Portable**:
  Works on Linux, macOS, Windows (compatible with MSVC, GCC, Clang)

## Getting Started

### Requirements

- C compiler supporting C11 (optional)
- POSIX or Win32 API for threading and console handling

### Installation

```sh
git clone https://github.com/0xA1M/clog.git
cd clog
```

Then copy `clog.h` into your project:

```c
#include "clog.h"
```

## Usage

### Basic Logging

```c
INFO("Server started on port %d", port);
WARN("Low disk space: %d%% remaining", percent);
ERROR("Failed to open file: %s", strerror(errno));
DEBUG("Cache hit for key '%s'", key);
TRACE("Entering function %s", __func__);
FATAL("Unexpected null pointer");
```

### Configuration

Set minimum log level (default is INFO):

```c
clog_set_level(CLOG_DEBUG);
```

Control color output (default is `CLOG_COLOR_AUTO`):

```c
clog_set_color_mode(CLOG_COLOR_NEVER);
clog_set_color_mode(CLOG_COLOR_ALWAYS);
clog_set_color_mode(CLOG_COLOR_ANSI);
clog_set_color_mode(CLOG_COLOR_WIN32);
```

Redirect log output (default is `stdout`):

```c
FILE *fp = fopen("app.log", "w");
clog_set_output(fp);
// ... logging calls ...
fclose(fp);
```

> ⚠️ **Note**: If you redirect output to a file, you should disable colors using `clog_set_color_mode(CLOG_COLOR_NEVER)` as ANSI escape codes may not render properly in files.
> ✅ **TODO**: Improve color handling for file outputs.

Cleanup (optional, automatically called via `atexit`):

```c
clog_cleanup();
```

### 编译期等级过滤

在 `#include "clog.h"` 之前定义 `CLOG_LOG_LEVEL`，低于它的日志从编译中移除 —— 调用和格式字符串都不会到达编译器：

```sh
gcc -DCLOG_LOG_LEVEL=CLOG_LEVEL_INFO app.c
```

```c
#define CLOG_LOG_LEVEL CLOG_LEVEL_WARN
#include "clog.h"
```

| `CLOG_LOG_LEVEL` | 保留的等级 |
| --- | --- |
| `CLOG_LEVEL_TRACE`（默认）、`0` | TRACE, DEBUG, INFO, WARN, ERROR, FATAL |
| `CLOG_LEVEL_DEBUG`、`1` | DEBUG, INFO, WARN, ERROR, FATAL |
| `CLOG_LEVEL_INFO`、`2` | INFO, WARN, ERROR, FATAL |
| `CLOG_LEVEL_WARN`、`3` | WARN, ERROR, FATAL |
| `CLOG_LEVEL_ERROR`、`4` | ERROR, FATAL |
| `CLOG_LEVEL_FATAL`、`5` | FATAL |
| `6`（比 `CLOG_LEVEL_FATAL` 大） | 全部移除 |

一个只调用 `DEBUG` 和 `TRACE` 的文件，把这两级滤掉后，`-Os` 下 `.text` 从 1685 字节降到 85 字节，`.bss` 从 2882 字节降到 0 —— 静态格式化缓冲区随代码一起消失。

注意：

- 写 `CLOG_LEVEL_INFO`，不要写 `INFO`。`#if` 中未定义的标识符会被当作 `0`，结果是所有等级都保留，编译器不报错。加 `-Wundef` 才能发现。
- 与 `clog_set_level(...)` 相互独立，后者仍在运行时过滤。两者同时生效时取更严的那个。
- 被滤掉的调用，其实参不求值，`DEBUG("v=%d", compute())` 里的副作用会一起消失。
- `LOG(level, ...)` 的等级是运行时值，永远不参与编译期过滤。
- 建议从构建系统传入该宏，保证每个编译单元取值一致。

## API Reference

```c
void clog_set_level(clog_level_t level);
void clog_set_color_mode(clog_color_mode_t mode);
void clog_set_output(FILE *fp);
void clog_cleanup(void);
void clog_signal_log(const char *message); // Async-signal-safe, minimal logging to stderr

enum clog_level_t {
    CLOG_TRACE, CLOG_DEBUG, CLOG_INFO,
    CLOG_WARN,  CLOG_ERROR, CLOG_FATAL
};

enum clog_color_mode_t {
    CLOG_COLOR_AUTO, CLOG_COLOR_NEVER,
    CLOG_COLOR_ALWAYS, CLOG_COLOR_ANSI,
    CLOG_COLOR_WIN32
};
```

All log macros expand to `clog_log(...)` and automatically capture file, line, and function.

## Building & Testing

The test suite lives under `tests/` and covers:

* Basic functionality and level filtering
* Memory-safety and buffer limits
* Signal-handler safety
* Thread-safety (requires pthreads)
* Performance benchmarks
* Configuration and output redirection

### Build & Run

```sh
make test
```

## Compatibility

✅ `clog` is **tested on Linux** (x86\_64) with:

* **GCC**, **Clang** compilers
* **glibc** and **musl** C libraries
* Both **multi-threaded** and **single-threaded** builds
* Modern **terminal emulators** with ANSI support (GNOME Terminal, Alacritty, etc.)

⚠️ Support for **Windows** is implemented but not yet fully tested in production.
If you use it on Windows, feedback and patches are welcome!

## Examples

See `tests/runner.c` for full test coverage and usage examples.

## Contributing

1. Fork the repo
2. Create your branch: `git checkout -b feature/foo`
3. Commit your changes
4. Push: `git push origin feature/foo`
5. Open a Pull Request

✅ Please include tests and make sure all tests pass before submitting.

## License

MIT License. See [LICENSE](LICENSE) for details.
