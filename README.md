# async-ping

Async ICMP ping library for Rust, built on [Tokio](https://tokio.rs/) and raw sockets.

Supports ICMPv4 and ICMPv6. Requires the process to have permission to open raw sockets (either `CAP_NET_RAW` on Linux or running as root).

---

## Library

### Cargo.toml

```toml
[dependencies]
async-ping = { path = "..." }
tokio = { version = "1", features = ["rt", "macros"] }
```

### High-level API

`ping` resolves the destination, opens the appropriate raw socket, sends `count` ICMP echo requests, and returns aggregate statistics.

```rust,no_run
use std::time::Duration;
use async_ping::ping;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let stats = ping("0.0.0.0", "8.8.8.8", 5, Duration::from_secs(1), 64).await?;
    println!(
        "{} tx / {} rx, rtt min/avg/max = {:.3}/{:.3}/{:.3} ms",
        stats.packets_tx,
        stats.packets_rx,
        stats.rtt_min.as_secs_f64() * 1000.0,
        stats.rtt_avg.as_secs_f64() * 1000.0,
        stats.rtt_max.as_secs_f64() * 1000.0,
    );
    Ok(())
}
```

`ping` accepts any type that implements `ToIpAddr` for both `src` and `dest`: `Ipv4Addr`, `Ipv6Addr`, `IpAddr`, `&str`, or `String`.

The `size` parameter is the total ICMP payload size in bytes. The first 8 bytes of the payload are reserved for an internal timestamp used to measure RTT; `size` must therefore be greater than 8.

#### PingStats fields

| Field          | Description                                      |
|----------------|--------------------------------------------------|
| `packets_tx`   | Number of echo requests sent                     |
| `packets_rx`   | Number of echo replies received                  |
| `rtt_min`      | Minimum RTT across received replies              |
| `rtt_avg`      | Mean RTT                                         |
| `rtt_max`      | Maximum RTT                                      |
| `rtt_std_dev`  | Population standard deviation of RTT samples     |

Probes that time out are counted in `packets_tx` but not `packets_rx`. They do not cause the function to return an error.

---

### Low-level API

For per-probe control, use `IcmpSocket` together with `send_icmp_echo_v4` / `send_icmp_echo_v6` directly.

```rust,no_run
use std::{net::Ipv4Addr, time::Duration};
use async_ping::{IcmpSocket, generate_payload, send_icmp_echo_v4};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let sock = IcmpSocket::bind(Ipv4Addr::UNSPECIFIED).await?;
    sock.connect("8.8.8.8").await?;

    let payload = generate_payload(56); // 56 bytes of application data
    let reply = send_icmp_echo_v4(&sock, &payload, 1, Duration::from_secs(5)).await?;

    println!(
        "reply from {}: seq={} ttl={} rtt={:.3} ms",
        reply.src_addr,
        reply.seq,
        reply.ttl,
        reply.rtt.as_secs_f64() * 1000.0
    );
    Ok(())
}
```

`send_icmp_echo_v4` / `send_icmp_echo_v6` each:
1. Build and send an ICMP echo request with an embedded timestamp.
2. Loop reading from the socket, filtering by type, ID, and sequence, until a matching reply arrives or the timeout elapses.
3. Return the reply metadata on success, or `ErrorKind::TimedOut` on timeout.

---

## Binary (`ping`)

A command-line ping utility is included behind the `bin` feature flag.

### Build

```sh
cargo build --release --features bin
```

The binary is placed at `target/release/ping`.

### Usage

```text
ping <destination> [-c count] [-s size] [-W timeout_secs]
```

| Flag | Default | Description                          |
|------|---------|--------------------------------------|
| `-c` | 5       | Number of echo requests to send      |
| `-s` | 56      | Total ICMP payload size (must be > 8)|
| `-W` | 1       | Per-probe timeout in seconds         |

### Example output

```text
$ ping 127.0.0.1 -c 5 -s 1500
PING 127.0.0.1 (127.0.0.1): 1500 data bytes
1508 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.106 ms
1508 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.139 ms
1508 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.102 ms
1508 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.152 ms
1508 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.132 ms

--- 127.0.0.1 ping statistics ---
5 packets transmitted, 5 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.102/0.126/0.152/0.019 ms
```

The reported byte count (`1508`) is ICMP header (8) + payload (`size`).

---

## Permissions

Raw sockets require elevated privileges:

**Linux** — grant `CAP_NET_RAW` to the binary:
```sh
sudo setcap cap_net_raw+ep target/release/ping
```

Or run with `sudo`.

**macOS / BSD** — run as root or use `sudo`.

---

## Dependencies

| Crate      | Purpose                                              |
|------------|------------------------------------------------------|
| `tokio`    | Async runtime, `AsyncFd` for non-blocking socket I/O |
| `socket2`  | Raw socket creation and `recvmsg` for ancillary data |
| `libc`     | `cmsghdr` / `CMSG_*` for IPv6 hop-limit extraction   |

## Minimum Rust version

1.84
