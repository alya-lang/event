# event

[![CI](https://github.com/alya-lang/event/actions/workflows/ci.yml/badge.svg)](https://github.com/alya-lang/event/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/alya-lang/event?color=blue&label=License)](LICENSE)
[![Alya](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fevent%2Fmain%2Falya.toml&query=%24.package.alya-version&label=Alya&color=orange&prefix=%3E%3D)](https://github.com/alya-lang/alya)
[![Package Version](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fevent%2Fmain%2Falya.toml&query=%24.package.version&label=Version&color=brightgreen)](alya.toml)

High-performance single-threaded reactor event loop, high-resolution timers, non-blocking socket I/O multiplexer, stream ByteBuffer, and decoupled EventEmitter for the Alya programming language.

---

## 🌟 Features

- ⚡ **Reactor Event Loop Engine**: Single-threaded non-blocking event loop capable of executing 2,500,000+ ticks/sec with dynamic sleep calculation.
- ⏱️ **High-Resolution Timer Queue**: Millisecond-accurate one-shot timeouts (`set_timeout`) and repeating intervals (`set_interval`) with deadline scheduling.
- 🌐 **Non-Blocking Socket Demultiplexer**: Cross-platform socket event multiplexer (`tcp_poll`) handling concurrent connections without blocking execution.
- 🌊 **Stream ByteBuffer**: Chunked zero-copy byte stream assembler with line-delimited reading (`buffer_read_line`) for HTTP, NDJSON, and custom text protocols.
- 📢 **Decoupled EventEmitter**: Blazing fast publish-subscribe pattern (4,000,000+ dispatches/sec) supporting persistent (`emitter_on`) and one-time (`emitter_once`) handlers.
- 🛡️ **Zero External C Dependencies (Phase 1)**: Pure Alya implementation leveraging `std/net` and `clock_ms()`, ready for future epoll/IOCP/kqueue native acceleration.
- 🧪 **100% Test Coverage**: Complete test suites for core lifecycle, timers, socket multiplexing, buffer operations, and event emissions.

---

## 📁 Project Architecture

```
event/
├── alya.toml               # Package manifest
├── src/
│   ├── lib.alya            # Public API facade & convenience aliases
│   ├── types.alya          # Core struct definitions (EventLoop, Timer, IoWatcher, ByteBuffer, EventEmitter)
│   ├── core/
│   │   ├── loop.alya       # Event loop tick, dynamic wait capping, event collection
│   │   ├── timer.alya      # High-res timer queue, deadline calculation, interval rescheduling
│   │   ├── watcher.alya    # Socket I/O watcher registry & handle management
│   │   └── poller.alya     # Cross-platform socket multiplexer using std/net
│   ├── emitter/
│   │   └── emitter.alya    # Decoupled publish-subscribe event emitter
│   └── stream/
│       └── buffer.alya     # Chunked stream byte buffer & line extractor
├── examples/
│   └── demo.alya           # Full runnable showcase demo (timers, sockets, buffer, emitter)
├── tests/
│   ├── test_basic.alya     # Core loop, timer cancel, watcher, and buffer tests (31 tests)
│   ├── test_timers.alya    # High-resolution timeout and interval test suite (15 tests)
│   ├── test_emitter.alya   # EventEmitter publish/subscribe test suite (17 tests)
│   └── test_poll.alya      # Real non-blocking TCP socket event polling test suite (16 tests)
└── benches/
    └── bench_basic.alya    # Performance micro-benchmarks (2.5M+ loop ticks/sec)
```

---

## 📦 Installation

Add `event` to the `[dependencies]` section in your `alya.toml`:

```toml
[dependencies]
event = { git = "https://github.com/alya-lang/event", branch = "main" }
```

Or install it directly using the Alya package CLI:

```bash
alyac add event --git https://github.com/alya-lang/event --branch main
alyac install
```

---

## 🚀 Quick Start

### 1. High-Resolution Timers & Intervals

```alya
import "event" as ev

let loop = ev::loop()

# Schedule a one-shot timeout (fires after 100ms)
ev::set_timeout(loop, 100, "cache_flush", null)

# Schedule a recurring interval (fires every 50ms)
let interval_id = ev::set_interval(loop, 50, "heartbeat", 42)

# Poll the event loop
while ev::loop_is_running(loop)
    let events = ev::loop_tick(loop, 50)
    let i = 0
    while i < len(events)
        let e = events[i]
        say "Fired event tag=" + e.tag + " type=" + ev::event_type_name(e.event_type)
        i += 1
    end
end
```

### 2. Non-Blocking TCP Socket Demultiplexing

```alya
import "std/net"
import "event" as ev

let loop = ev::loop()
let server = tcp_listen(8080, 128)
tcp_set_nonblocking(server, 1)

# Watch server for readable incoming connections
ev::watch_read(loop, server, "server_accept", null)

# Process incoming network events without blocking
let events = ev::loop_tick(loop, 100)
let i = 0
while i < len(events)
    let e = events[i]
    if e.tag == "server_accept"
        let client = tcp_accept(server)
        tcp_set_nonblocking(client, 1)
        ev::watch_read(loop, client, "client_read", null)
    end
    i += 1
end
```

### 3. Decoupled EventEmitter (Pub/Sub)

```alya
import "event" as ev

let em = ev::emitter()

# Register persistent and one-time listeners
ev::emitter_on(em, "order_created", "send_notification")
ev::emitter_once(em, "order_created", "first_purchase_gift")

# Emit event with payload data
let triggered = ev::emitter_emit(em, "order_created", {"order_id": 1001})
# -> Dispatches to "send_notification" and "first_purchase_gift"
```

---

## 📖 API Reference

### EventLoop Core

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `loop()` | None | `EventLoop` | Creates a new initialized `EventLoop` instance. |
| `loop_tick(loop, max_wait_ms)` | `loop`, `max_wait_ms = 100` | `array` | Executes a single reactor cycle (dynamic sleep, poll I/O, collect timers) and returns triggered `EventNotification` records. |
| `loop_poll(loop, max_wait_ms)` | `loop`, `max_wait_ms = 100` | `array` | Alias for `loop_tick`. Polls the event loop for up to `max_wait_ms`. |
| `loop_is_running(loop)` | `loop` | `integer` | Returns `1` if the loop has active timers or watchers and is not stopped, `0` otherwise. |
| `loop_active_count(loop)` | `loop` | `integer` | Returns total active handles (active timers + active socket watchers). |
| `loop_stop(loop)` | `loop` | `void` | Requests the event loop to stop processing further ticks. |
| `loop_reset(loop)` | `loop` | `void` | Clears all registered timers and socket watchers, resetting the loop. |

### Timers & Intervals

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `set_timeout(loop, delay_ms, tag, data)` | `loop`, `delay_ms`, `tag = ""`, `data = null` | `integer` | Schedules a one-shot timer to expire after `delay_ms`. Returns assigned timer ID. |
| `set_interval(loop, interval_ms, tag, data)` | `loop`, `interval_ms`, `tag = ""`, `data = null` | `integer` | Schedules a repeating timer every `interval_ms`. Returns assigned timer ID. |
| `clear_timer(loop, timer_id)` | `loop`, `timer_id` | `integer` | Cancels an active timeout or interval. Returns `1` if cancelled, `0` if not found. |

### Socket I/O Watchers

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `watch_read(loop, fd, tag, data)` | `loop`, `fd`, `tag = ""`, `data = null` | `integer` | Registers a non-blocking socket to watch for readable incoming data (`EVENT_READ`). |
| `watch_write(loop, fd, tag, data)` | `loop`, `fd`, `tag = ""`, `data = null` | `integer` | Registers a non-blocking socket to watch for writable readiness (`EVENT_WRITE`). |
| `unwatch(loop, fd)` | `loop`, `fd` | `integer` | Deregisters all watchers associated with socket `fd`. Returns `1` if removed, `0` if not found. |

### Stream ByteBuffer

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `buffer()` | None | `ByteBuffer` | Creates a new empty `ByteBuffer` instance. |
| `buffer_append(buf, chunk)` | `buf`, `chunk: string` | `void` | Appends a text/byte chunk to the stream buffer. |
| `buffer_read_line(buf)` | `buf` | `[string, integer]` | Extracts the next `\n` or `\r\n` delimited line. Returns `[line, 1]` on success, `["", 0]` if incomplete. |
| `buffer_len(buf)` | `buf` | `integer` | Returns total buffered bytes remaining. |
| `buffer_clear(buf)` | `buf` | `void` | Discards all buffered chunks. |

### EventEmitter (Publish/Subscribe)

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `emitter()` | None | `EventEmitter` | Creates a new `EventEmitter` instance. |
| `emitter_on(em, event, tag)` | `em`, `event: string`, `tag: string` | `integer` | Registers a persistent listener tag for an event. |
| `emitter_once(em, event, tag)` | `em`, `event: string`, `tag: string` | `integer` | Registers a one-time listener tag auto-removed after firing once. |
| `emitter_off(em, event, tag)` | `em`, `event: string`, `tag: string` | `integer` | Removes a listener by tag or ID. Returns `1` if removed, `0` if not found. |
| `emitter_emit(em, event, data)` | `em`, `event: string`, `data = null` | `array` | Dispatches event to all active listeners and returns triggered handler tags. |
| `emitter_listener_count(em, event)` | `em`, `event: string` | `integer` | Returns count of active listeners for an event. |
| `emitter_clear(em, event)` | `em`, `event = null` | `integer` | Clears all listeners for an event or all events if `event == null`. |

### Event Types & Constants

| Constant Function | Value | Description |
|---|---|---|
| `EVENT_NONE()` | `0` | No event occurred / idle state. |
| `EVENT_READ()` | `1` | Socket is ready to be read from. |
| `EVENT_WRITE()` | `2` | Socket is ready to write data without blocking. |
| `EVENT_TIMER()` | `3` | Scheduled timer or interval expired. |
| `EVENT_ERROR()` | `4` | Socket error or exception occurred. |
| `EVENT_CLOSE()` | `5` | Remote peer closed connection / EOF. |
| `event_type_name(t)` | String | Converts integer event constant into human-readable string (`"READ"`, `"TIMER"`, etc.). |

---

## 🧪 Running Tests & Benchmarks

Run the complete automated test suite:

```bash
alyac run tests/test_basic.alya
alyac run tests/test_timers.alya
alyac run tests/test_emitter.alya
alyac run tests/test_poll.alya
```

Run the performance micro-benchmarks:

```bash
alyac run benches/bench_basic.alya
```

Run the comprehensive feature demonstration:

```bash
alyac run examples/demo.alya
```

---

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository and clone it locally
2. Install dependencies:
   ```bash
   alyac install
   ```
3. Create your feature branch (`git checkout -b feature/my-feature`)
4. Verify tests and formatting before opening a PR:
   ```bash
   alyac test
   alyac fmt . --check
   ```
5. Commit your changes (`git commit -m "feat: add feature"`) and open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.