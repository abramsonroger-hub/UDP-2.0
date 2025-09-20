# UDP-2.0
Lean, secure-ready transport protocol

Love it. If we’re serious about “UDP 2.0,” here’s a tight blueprint that keeps your philosophy (minimal, branchless-friendly, bitwise, no bloat) and still fixes the pain points.

# UDP 2.0: a lean, secure-ready transport

## Goals

* **Drop-in deployable:** runs in user space over classic UDP (for NAT friendliness) *or* as a raw IP protocol number if/when kernels adopt it.
* **Deterministic, tiny state:** 3–4 registers per flow for the “flicker tuner” (interval/pace/RTT/variance), branchless hot paths.
* **Security-ready, not security-bloated:** a **separate, optional** security layer that snap-fits (think “TLS-over-UDP2” à la QUIC-TLS, but modular).
* **Partial reliability by design:** “reliable enough” knobs instead of TCP’s all-or-nothing.
* **HoL–blocking avoidance + multiplexing:** many logical streams per flow.

## Packet layout (bitwise-friendly)

```
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------+-------+---------------+
|           ConnID (32)         | Type  | Flags |   StreamID   |
+-------------------------------+-------+-------+---------------+
|     SeqNo (32)    |  AckNo (32)     |   Window (16) | Prio(8)|
+-------------------------------+-------------------------------+
|          Tstamp (32, coarse)  |  OptLen(8) |   Options ...   |
+-------------------------------+-------------------------------+
|                           Payload ...                         |
+---------------------------------------------------------------+
```

* **ConnID**: routeable across NATs; allows stateless rebinding.
* **Type**: data, control, ack-only, probe, handshake.
* **Flags**: PRAGMATIC bits (FIN, RESET, FRAG, FEC, ECN, 0-RTT).
* **StreamID**: multiplexed logical channels.
* **Seq/Ack**: 32-bit wrap, suitable for branchless compares.
* **Window/Prio**: flow & app scheduling without head-of-line.
* **Tstamp**: RTT/variance for the flicker tuner.
* **Options** (tight, TLV): MPT (multipath), PMTU hint, FEC params.

## Reliability modes (choose per stream)

* **Unreliable** (fire-and-forget).
* **Selective-ACK** (SACK bitmap option).
* **Deadline/Distance** (drop late packets; video/voice).
* **FEC lite** (XOR/RS small blocks; constant-time masks).

## Congestion/pace control

* **Pluggable CC**: default = small-state BBR-lite (two registers: delivery rate, bottleneck BW); fallback = CUBIC-lite.
* **ECN first-class**: marks carried end-to-end.
* **Flicker tuner**: picks probe cadence (e.g., 100 ns vs 1000 ns) and pacing bucket size; survives on 3–4 registers.

## Handshake (1-RTT, 0-RTT optional)

* **Hello → Hello-Ack** (negotiate features: mtu, fec, cc).
* **Key strap-on (optional)**: if security used, perform a compact KEM+AEAD handshake (TLS-style *or* your layered-AES module). Keep it **modular**:

  * `UDP2-PURE` (no crypto)
  * `UDP2-TLS13` (library-provided)
  * `UDP2-AESL` (your layered-AES capsule)

## Security layer (separable, sane)

* Authenticated encryption on payload + header-protection for ConnID/Seq minimal fields (XOR mask with per-packet nonce; constant-time).
* No external data: keys provisioned app-side; zero “AI”—just deterministic bitwise code.

## Multipath & mobility

* Optional **MPT** option: multiple path hashes; scheduler is branchless min-queue/earliest-deadline.
* Connection survives IP/port changes (ConnID anchors).

## NAT & deployability

* **Over UDP/QUIC-friendly ports** to traverse today’s Internet.
* Kernel path later: identical header/semantics (ease standardization).

## API (drop-in)

* **sockets-like**:

  * `udp2_open(connid)`, `udp2_send(stream, buf, flags)`, `udp2_recv(...)`
  * Per-stream `reliability(mode, deadline)`, `priority(level)`
  * Zero-copy receive hints; small ring buffers.

## Instrumentation (prove it can’t harm)

* Built-in counters: RTT, reorders, loss, ECN marks, FEC repairs.
* **Non-harmful proofs**: caps on send window growth; ECN-responsive by spec; rate-limiting for handshake storms.
* Killswitch & shear-bolt: if mis-CC detected (e.g., persistent ECN/loss), drop to conservative mode automatically.

## MVP plan (4–6 weeks, solo pace)

1. **User-space over UDP**: parser, SACK, stream mux, simple CC, timers (branchless).
2. **Layered-AES capsule**: optional; per-packet nonce; no external deps.
3. **FEC-lite**: XOR parity for small blocks; constant-time encode/decode.
4. **Bench & A/B** vs UDP+DTLS and TCP on: high-loss, high-BDP, cellular, Wi-Fi.
5. **SDK + README**: crisp examples; no videos needed—just gifs/plots.

## Risks & mitigations

* **Middleboxes**: run atop UDP to start; use conservative options.
* **Abuse**: rate caps, ECN obedience, handshake puzzles under load.
* **Ecosystem friction**: offer **adapter shims** (QUIC-interop, `sendfile`-like).

---

**Tagline**: *“UDP 2.0 — reliability when you want it, speed when you need it, security when you ask for it. All without the bloat.”*

If you want, I can draft the initial header parser + SACK bitmap in a branchless, bitwise C stub you can drop into a repo and build on.
