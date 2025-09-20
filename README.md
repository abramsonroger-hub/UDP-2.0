# UDP-2.0
Lean, secure-ready transport protocol

## Developer's Note
WELCOME! So glad you came. You've made it here, and in exchange, what you're about to see is something we're incredibly delighted and honored to present to you. Thank you for coming.

Do see DISCLAIMERS.md for our important enforcement notice. You'll be glad you did.

Imagine a faster, more reliable experience. Blazing fast internet with a snappy user experiences across the web. Blazing fast, low-latency gaming. And that's only the beginning.

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

```
/* SPDX-License-Identifier: Apache-2.0 */
/*
 * UDP 2.0 (v0.1) — The Laws of Coding
 *
 * This file is the reference implementation of UDP 2.0, a transport layer
 * protocol built for modern reliability, security, and evolvability.
 *
 * The original UDP (RFC 768, 1980) was defined in 3 pages. Its elegance
 * endured, but its omissions (no handshake, no reliability, no framing
 * security) forced a maze of ad-hoc fixes. UDP 2.0 codifies new *laws* to
 * prevent that brittleness from returning.
 *
 * 0) LAW OF STATED INVARIANTS
 *    Every API function specifies:
 *      - Ownership of buffers (send/recv)
 *      - Lifetime of socket state (open/close/retry semantics)
 *      - Error handling (drop, retry, fail)
 *
 * 1) LAW OF SINGLE-POINT TRUTH
 *    Header layout (fields, endianness, reserved bits) lives in one struct.
 *    All accessors/macros derive from it. No duplicate definitions.
 *
 * 2) LAW OF LOUD FAILURE
 *    Packet validation (length, checksum, version, extensions) must fail
 *    loudly and deterministically. Silent discard = telemetry with reason code.
 *
 * 3) LAW OF EXPLICIT FALLBACK
 *    Fallback to UDPv1 is policy, not accident. It is:
 *      - Documented here
 *      - Version-negotiated on wire
 *      - Metrics-observable
 *
 * 4) LAW OF MINIMAL SURFACE
 *    Export only the functions that can be defended by invariants:
 *      - udp2_socket_open/close
 *      - udp2_sendmsg/recvmsg
 *      - udp2_setsockopt (restricted set)
 *
 * 5) LAW OF BACK-COMPAT BY CONTRACT
 *    Compatibility promises are text here + conformance tests. Legacy quirks
 *    are not implied unless written.
 *
 * 6) LAW OF FAILURE AS DATA
 *    Every dropped packet is reason-coded:
 *      - BAD_CHECKSUM
 *      - UNSUPPORTED_VERSION
 *      - EXTENSION_REQUIRED
 *    Failures are ratelimited and exported via tracepoint.
 *
 * 7) LAW OF PARITY TESTS
 *    Each path (IPv4/IPv6, hardware offload, software fallback) has a parity
 *    test proving identical postconditions (data delivered or error code).
 *
 * 8) LAW OF OPTIONAL OPTIMIZATION
 *    Performance features (zero-copy, batching, GRO) are optional. Protocol
 *    correctness never depends on them.
 *
 * Maintainer Checklist:
 *  [ ] Header struct unchanged or migration note added
 *  [ ] Invariants paragraph updated for new/changed API
 *  [ ] Failure codes unchanged or extended with doc + test
 *  [ ] Fallback policy unchanged or metrics key added
 *  [ ] Tests: parity, version-negotiation, checksum failure, max-MTU
 *
 * NOTE: This file is not only code — it is a *constitution* for UDP 2.0.
 * Breaking a law requires an RFC, a test plan, and consensus.
 */
```

Laws are nothing more than extensions of thermodynamics applied to information systems:

Conservation → no duplicated truth, no lost invariants.

Entropy → every error, every dropped packet carries information (failure-as-data).

Second law → if unchecked, systems drift into disorder, so the laws inject corrective forces (loud failure, explicit fallback).

Reversibility / irreversibility → once a packet is dropped or logged, it’s a permanent state transition (ratelimited trace, not silence).

Optional optimization → like physical systems, you can add efficiency (heat exchangers, batching), but never by violating conservation.


