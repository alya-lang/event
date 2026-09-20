# event

[![CI](https://github.com/alya-lang/event/actions/workflows/ci.yml/badge.svg)](https://github.com/alya-lang/event/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/alya-lang/event?color=blue&label=License)](LICENSE)
[![Alya](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fevent%2Fmain%2Falya.toml&query=%24.package.alya-version&label=Alya&color=orange&prefix=%3E%3D)](https://github.com/alya-lang/alya)
[![Package Version](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fevent%2Fmain%2Falya.toml&query=%24.package.version&label=Version&color=brightgreen)](alya.toml)

High-performance single-threaded reactor event loop, high-resolution timers, non-blocking socket I/O multiplexer, stream ByteBuffer, and decoupled EventEmitter for the Alya programming language.

---

## 🌟 Features

- ⚡ **Reactor Event Loop Engine**: Single-threaded non-blocking event loop capable of executing 2,500,000+ ticks/sec with dynamic sleep calculation.
- 🚀 **OS Kernel Multiplexing (`Lib/uv`)**: High-performance I/O multiplexer powered by `Lib/uv` (`epoll` on Linux, `kqueue` on macOS/BSD, `WSAPoll` on Windows) with automatic fallback to `std/net`.
- 🖥️ **Event-Driven `TcpServer` & `TcpClient`**: High-level stream abstractions wrapping non-blocking raw socket descriptors with automatic lifecycle management.
- 🌊 **Buffered `StreamReader` & `RingBuffer`**: High-speed circular ring buffer and delimiter message framer (`read_line`, `read_until`, `read_bytes`) processing 1,250,000+ frames/sec.
- 🛑 **Backpressure & Flow Control**: Output buffer threshold enforcement (`high_water_mark`, `on_drain`) and non-blocking read pause/resume controls.
- ⏱️ **High-Resolution Timer Queue**: Millisecond-accurate one-shot timeouts (`set_timeout`) and repeating intervals (`set_interval`) with deadline scheduling.
- 🌐 **Non-Blocking Socket Demultiplexer**: Multi-socket concurrent I/O handling thousands of simultaneous connections without blocking execution.
- 📡 **Cross-Platform Signal Handling**: Seamless process signal watching (`watch_signal` / `unwatch_signal`) for lifecycle events (`SIGINT`, `SIGTERM`).
- 📢 **Decoupled EventEmitter**: Blazing fast publish-subscribe pattern (4,000,000+ dispatches/sec) supporting persistent (`emitter_on`) and one-time (`emitter_once`) handlers.
- 🧪 **100% Test Coverage**: Complete test suites for core lifecycle, multiplexing, servers, streams, framing, and buffers (9 test suites, 100+ assertions).

---

## 📁 Project Architecture

```
event/
├── alya.toml               # Package manifest (Lib/uv dependency)
├── src/
│   ├── lib.alya            # Public API facade & convenience aliases
│   ├── types.alya          # Core structs (EventLoop, TcpServer, TcpStream, StreamReader, RingBuffer...)
│   ├── core/
│   │   ├── loop.alya       # Event loop tick, dynamic wait capping, event collection, backend discovery
│   │   ├── timer.alya      # High-res timer queue, deadline calculation, interval rescheduling
│   │   ├── watcher.alya    # Socket I/O watcher registry & handle management
│   │   └── poller.alya     # Cross-platform socket multiplexer (Lib/uv with std/net fallback)
│   ├── emitter/
│   │   └── emitter.alya    # Decoupled publish-subscribe event emitter
│   ├── stream/
│   │   ├── buffer.alya     # Chunked stream byte buffer & line extractor
│   │   ├── ring.alya       # High-speed circular ring buffer
│   │   ├── reader.alya     # Message framing (read_line, read_until, read_bytes)
│   │   └── stream.alya     # High-level TcpStream with flow control & backpressure
│   └── net/
│       ├── server.alya     # Event-driven TcpServer factory & connection management
│       └── client.alya     # Non-blocking TcpClient connection helper
├── examples/
│   └── demo.alya           # Full runnable showcase demo (timers, sockets, streams, server, framing)
├── tests/
│   ├── test_basic.alya     # Core loop, timer cancel, watcher, and buffer tests
│   ├── test_timers.alya    # High-resolution timeout and interval test suite
│   ├── test_emitter.alya   # EventEmitter publish/subscribe test suite
│   ├── test_poll.alya      # Non-blocking TCP socket event polling test suite
│   ├── test_run.alya       # Continuous event loop execution test suite
│   ├── test_multiplexer.alya # Multi-client kernel multiplexer test suite
│   ├── test_reader.alya    # RingBuffer and StreamReader framing test suite
│   ├── test_stream.alya    # TcpStream flow control & backpressure test suite
│   └── test_net.alya       # End-to-end TcpServer & TcpClient integration test suite
└── benches/
    └── bench_basic.alya    # Performance micro-benchmarks (7 methods, 1M+ ops/sec)
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
alya add event --git https://github.com/alya-lang/event --branch main
alya install
```

---

## 🚀 Quick Start

```alya
import "event" as ev

function main()
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
end

main()
```

### Non-Blocking TCP Socket Demultiplexing

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

### Decoupled EventEmitter (Pub/Sub)

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

### High-Level Event-Driven TcpServer & TcpClient

```alya
import "event" as ev

let loop = ev::loop()

# Start high-level event-driven TCP server
let server = ev::server(loop, "127.0.0.1", 8080, function(srv, client)
    say "Client connected: " + client.peer_addr
    ev::stream_on_line(client, function(stream, line)
        say "Received: " + line
        ev::stream_write(stream, "ECHO:" + line + "\n")
    end)
end)

# Connect client non-blockingly
let client = ev::client(loop, "127.0.0.1", 8080)
ev::stream_on_line(client, function(stream, line)
    say "Server replied: " + line
end)

ev::stream_write(client, "Hello World\n")
loop.run()
```

### StreamReader & RingBuffer Message Framing

```alya
import "event" as ev

let reader = ev::reader("\n")

# Feed arbitrary incoming packet chunks
reader.feed("POST /v1/telemetry HTTP/1.1\r\nHost: api.alya.org\r\n\r\n")

# Extract delimited lines and payload frames
let line1 = reader.read_line() # -> ["POST /v1/telemetry HTTP/1.1", 1]
let line2 = reader.read_line() # -> ["Host: api.alya.org", 1]
```

---

## 📖 API Reference

### EventLoop Core

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `loop(use_uv = true)` | `use_uv = true` | `EventLoop` | Creates a new initialized `EventLoop` instance with native `Lib/uv` kernel multiplexing (or `std/net` fallback if false). |
| `loop_backend(loop)` | `loop` | `string` | Returns active multiplexer backend (`"epoll"`, `"WSAPoll"`, `"kqueue"`, or `"std/net"`). |
| `loop_tick(loop, max_wait_ms)` | `loop`, `max_wait_ms = 100` | `array` | Executes a single reactor cycle (dynamic sleep, poll I/O, collect timers) and returns triggered `EventNotification` records. |
| `loop_poll(loop, max_wait_ms)` | `loop`, `max_wait_ms = 100` | `array` | Alias for `loop_tick`. Polls the event loop for up to `max_wait_ms`. |
| `loop_run(loop, max_wait_ms, on_event)` | `loop`, `max_wait_ms = 100`, `on_event = null` | `integer` | Runs the event loop continuously until all timers and watchers complete or stop is requested. Invokes `on_event(ev)` for each event. Returns total events processed. |
| `loop_is_running(loop)` | `loop` | `integer` | Returns `1` if the loop has active timers or watchers and is not stopped, `0` otherwise. |
| `loop_active_count(loop)` | `loop` | `integer` | Returns total active handles (active timers + active socket watchers). |
| `loop_stop(loop)` | `loop` | `void` | Requests the event loop to stop processing further ticks. |
| `loop_reset(loop)` | `loop` | `void` | Clears all registered timers and socket watchers, resetting the loop. |
| `loop_free(loop)` | `loop` | `void` | Frees and resets all resources in the event loop. |

### High-Level TCP Server & Client

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `server(loop, host, port, on_connect, on_error, backlog)` | `loop`, `host`, `port`, `on_connect`, `on_error = null`, `backlog = 128` | `TcpServer` | Creates and binds event-driven TCP server. Dispatches incoming connections to `on_connect(server, client_stream)`. |
| `client(loop, host, port, on_connect, on_error)` | `loop`, `host`, `port`, `on_connect = null`, `on_error = null` | `TcpStream` | Initiates non-blocking TCP client connection. |
| `tcp_server_close(server)` | `server` | `integer` | Stops listening, closes all connected client streams, and frees server handle. |
| `server.client_count()` | `server` | `integer` | Returns count of active connected clients. |
| `server.is_listening()` | `server` | `integer` | Returns `1` if server is actively listening, `0` otherwise. |

### TcpStream, Flow Control & Backpressure

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `stream(loop, fd, high_water_mark)` | `loop`, `fd`, `high_water_mark = 65536` | `TcpStream` | Wraps a non-blocking socket into an event-driven `TcpStream`. |
| `stream_write(stream, data)` | `stream`, `data` | `integer` | Writes non-blockingly. Returns `1` if accepted below high water mark, `0` if backpressure threshold exceeded, `-1` on error. |
| `stream_read(stream, max_bytes)` | `stream`, `max_bytes = 4096` | `string` | Reads incoming chunk, feeds stream reader, and dispatches callbacks. |
| `stream_pause(stream)` | `stream` | `integer` | Flow control: pauses reading from socket by removing readable watcher. |
| `stream_resume(stream)` | `stream` | `integer` | Flow control: resumes reading from socket. |
| `stream_close(stream)` | `stream` | `integer` | Closes socket and unregisters all loop watchers. |
| `stream_on_line(stream, callback)` | `stream`, `callback` | `TcpStream` | Registers line-delimited message callback: `function(stream, line)`. |
| `stream_on_data(stream, callback)` | `stream`, `callback` | `TcpStream` | Registers raw chunk callback: `function(stream, chunk)`. |
| `stream_on_close(stream, callback)` | `stream`, `callback` | `TcpStream` | Registers disconnect callback: `function(stream)`. |
| `stream_on_drain(stream, callback)` | `stream`, `callback` | `TcpStream` | Registers buffer drained callback: `function(stream)`. |
| `stream.is_paused()` | `stream` | `integer` | Returns `1` if stream reading is paused, `0` otherwise. |
| `stream.is_closed()` | `stream` | `integer` | Returns `1` if stream socket is closed, `0` otherwise. |
| `stream.buffered_bytes()` | `stream` | `integer` | Returns count of pending unsent bytes in write buffer. |

### StreamReader & RingBuffer

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `reader(delimiter, max_frame_size, capacity)` | `delim = "\n"`, `max_frame = 1MB`, `cap = 64KB` | `StreamReader` | Creates buffered stream reader with specified delimiter. |
| `reader.feed(chunk)` | `chunk` | `integer` | Feeds incoming raw chunk into internal buffer. |
| `reader.read_line()` | `reader` | `[string, integer]` | Extracts next line terminated by `\n` (stripping trailing `\r`). Returns `[line, 1]` or `["", 0]`. |
| `reader.read_until(delim)` | `delim` | `[string, integer]` | Extracts frame up to custom delimiter `delim`. |
| `reader.read_bytes(n)` | `n` | `[string, integer]` | Extracts fixed-length packet of exactly `n` bytes. |
| `reader.peek(n)` | `n` | `string` | Peeks up to `n` bytes without consuming. |
| `reader.available()` | `reader` | `integer` | Returns buffered bytes count. |
| `reader.clear()` | `reader` | `void` | Resets and discards buffered content. |
| `ring(capacity)` | `capacity = 65536` | `RingBuffer` | Creates a high-speed circular ring buffer. |
| `ring.write(chunk)` | `chunk` | `integer` | Writes chunk up to capacity. Returns bytes accepted or `-1` if full. |
| `ring.read(n)` | `n` | `string` | Consumes and returns up to `n` bytes. |
| `ring.free_space()` | `ring` | `integer` | Returns remaining capacity before full. |

### Timers & Intervals

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `set_timeout(loop, delay_ms, tag, data)` | `loop`, `delay_ms`, `tag = ""`, `data = null` | `integer` | Schedules a one-shot timer to expire after `delay_ms`. Returns assigned timer ID. |
| `set_interval(loop, interval_ms, tag, data)` | `loop`, `interval_ms`, `tag = ""`, `data = null` | `integer` | Schedules a repeating timer every `interval_ms`. Returns assigned timer ID. |
| `clear_timer(loop, timer_id)` | `loop`, `timer_id` | `integer` | Cancels an active timeout or interval. Returns `1` if cancelled, `0` if not found. |

### Socket I/O Watchers & Signals

| Function | Arguments | Returns | Description |
|---|---|---|---|
| `watch_read(loop, fd, tag, data)` | `loop`, `fd`, `tag = ""`, `data = null` | `integer` | Registers a non-blocking socket to watch for readable incoming data (`EventType.Read`). |
| `watch_readable(loop, fd, tag, data)` | `loop`, `fd`, `tag = ""`, `data = null` | `integer` | Alias for `watch_read`. Registers socket to watch for incoming data. |
| `watch_write(loop, fd, tag, data)` | `loop`, `fd`, `tag = ""`, `data = null` | `integer` | Registers a non-blocking socket to watch for writable readiness (`EventType.Write`). |
| `watch_writable(loop, fd, tag = "", data = null)` | `loop`, `fd`, `tag = ""`, `data = null` | `integer` | Alias for `watch_write`. Registers socket to watch for writable readiness. |
| `watch_signal(loop, signum, tag, data)` | `loop`, `signum`, `tag = ""`, `data = null` | `integer` | Registers an OS process signal to watch (e.g. `SIGINT`, `SIGTERM`). |
| `unwatch(loop, fd)` | `loop`, `fd` | `integer` | Deregisters all watchers associated with socket `fd`. Returns `1` if removed, `0` if not found. |
| `unwatch_signal(loop, signum)` | `loop`, `signum` | `integer` | Deregisters watcher for process signal `signum`. |

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

### Event Types & Enums

| Enum Variant | Value | Description |
|---|---|---|
| `EventType.None` | `0` | No event occurred / idle state. |
| `EventType.Read` | `1` | Socket is ready to be read from. |
| `EventType.Write` | `2` | Socket is ready to write data without blocking. |
| `EventType.Timer` | `3` | Scheduled timer or interval expired. |
| `EventType.Error` | `4` | Socket error or exception occurred. |
| `EventType.Close` | `5` | Remote peer closed connection / EOF. |
| `EventType.Signal` | `6` | Operating system process signal received. |
| `event_type_name(t)` | String | Converts integer event constant into human-readable string (`"READ"`, `"TIMER"`, `"SIGNAL"`, etc.). |

---

## 🧪 Running Tests & Benchmarks

Run the complete automated test suite:

```bash
alya test .
```

Or run individual test suites:

```bash
alya run tests/test_basic.alya
alya run tests/test_timers.alya
alya run tests/test_emitter.alya
alya run tests/test_poll.alya
alya run tests/test_run.alya
alya run tests/test_multiplexer.alya
alya run tests/test_reader.alya
alya run tests/test_stream.alya
alya run tests/test_net.alya
```

Run the performance micro-benchmarks:

```bash
alya run benches/bench_basic.alya
```

Run the comprehensive feature demonstration:

```bash
alya run examples/demo.alya
```

Check code formatting:

```bash
alya fmt . --check
```

Run static code linter:

```bash
alya lint . --check
```

---

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository and clone it locally
2. Install dependencies:
   ```bash
   alya install
   ```
3. Create your feature branch (`git checkout -b feature/my-feature`)
4. Verify tests and formatting before opening a PR:
   ```bash
   alya test
   alya fmt . --check
   ```
5. Commit your changes (`git commit -m "feat: add feature"`) and open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.