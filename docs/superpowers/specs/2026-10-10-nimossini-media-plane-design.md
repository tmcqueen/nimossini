# Nimossini Media Plane and PoC Control Plane: Design

Date: 2026-10-10
Status: Draft for review

## 1. Purpose and scope

Nimossini is an open-source media server written in Rust. It is the first
building block of a longer-term goal: an open-source, cloud-native
Communication-as-a-Service (CPaaS) platform.

The long-term platform has six subsystems. This spec covers only the first two.

| # | Subsystem | Status |
|---|---|---|
| 1 | Media plane (Nimossini) | In scope |
| 2 | Minimal control plane (proof of concept) | In scope, kept deliberately small |
| 3 | Public API (resource model, auth, multi-tenancy) | Deferred |
| 4 | Voice control plane (routing, IVR, queues, features) | Deferred |
| 5 | Scripting / execution environment | Deferred |
| 6 | Other channels (email, SMS, ...) | Deferred |

Subsystems 3-6 are constraints on this design, not deliverables. The
control-to-media gRPC contract is the interface they will build on, so it must
not need to change when they arrive.

The author is learning Rust and will implement most of the code. The
implementation plan is therefore written as small, ordered milestones with
tests first (see section 12).

## 2. Goals and non-goals

Goals (media plane):

- RTP to RTP, with and without transcoding
- RTP to WebRTC, with and without transcoding
- Speaker detection on all legs
- Local speech-to-text (STT) on all legs
- Play audio to one or more legs
- Shared audio sources (hold music)
- Conferencing with speaker detection and transcription, up to
  `MAX_LEGS_PER_BRIDGE` participants (default 32)
- Mid-call codec changes
- Mute, deafen, supervisor listen / whisper / barge
- Legal intercept (design and isolation properties proven in the PoC)

Non-goals for this spec:

- The deferred subsystems (3-6)
- Video
- A packet-forward fast path (every leg is decoded; revisit only if profiling
  demands it)
- Surviving a media-node crash (nodes are disposable; the control plane
  decides what to do)
- Legal intercept delivery standards (ETSI / CALEA mediation interfaces),
  warrant management, retention

Scale target: dozens of concurrent calls per node, conferences up to 32
participants, limited by measured CPU.

## 3. Architecture overview

```
 carrier / phones            WebRTC clients
        \                        /
         \                      /
   +------v--------------------v------+
   |   Control plane PoC (ctl-poc)     |  Rust SIP UAS/B2BUA
   |   SIP signalling, call policy     |  (Kamailio/OpenSIPS edge later)
   +----------------+------------------+
                    | gRPC (tonic)
   +----------------v------------------+
   |   Nimossini media server          |
   |   +-----------------------------+ |
   |   | core: matrix, mixer, taps   | |  pure Rust, no GStreamer
   |   +--------------+--------------+ |
   |   | edge: per-leg GStreamer     | |  RTP/SRTP/WebRTC, codecs
   |   +-----------------------------+ |
   +-----------------------------------+
```

Key decisions:

- The control plane owns SIP and call policy. Nimossini owns media, including
  SDP generation (see section 6).
- GStreamer is used only at the edges (one pipeline per leg). The mixing core
  is plain Rust.
- The control plane PoC is a Rust SIP UAS/B2BUA using a Rust SIP library. A
  short spike must confirm the library is adequate (INVITE/200/ACK, re-INVITE,
  BYE, a call to and from a real trunk). Fallback: FreeSWITCH as the SIP
  front end for the PoC, with no change to Nimossini's contract.
- Kamailio or OpenSIPS may later be added as a SIP edge proxy.

## 4. Core and dataflow

### 4.1 Components

- Leg edge (GStreamer): one pipeline per leg. Handles RTP/SRTP/WebRTC, jitter
  buffer, depayload, decode, resample to the internal format. Frames leave
  through `appsink`; mixed frames return through `appsrc` for encode, payload
  and send.
- Bridge core (pure Rust): owns the mixing matrix, gates, taps, sources and
  the 20 ms tick.
- Sources: file/prompt players and shared hold streams, as PCM producers in
  the core.
- Taps: VAD/level, STT feeds, intercept copy, as consumers of per-leg PCM at
  defined points.

### 4.2 Internal format

Mono, 16-bit PCM, 20 ms frames, one fixed internal sample rate. Proposed rate
is 16 kHz (suits Opus and STT). To be confirmed during the edge milestone.

### 4.3 The tick (every 20 ms)

1. Collect one frame from each leg's receive queue (silence/concealment if
   missing).
2. Run VAD and level; update the active-speaker ranking.
3. Hand each leg's frame to the taps (STT, intercept) before any gate.
4. Apply mute gates, compute each sink's mix from the matrix (N-1, top-K
   active speakers).
5. Apply deafen gates and push each result to its leg's send queue.

The tick is a deterministic function of frames in to frames out, so the core
is testable without network or GStreamer.

### 4.4 Mixing matrix

The bridge is a source-to-sink matrix with a gain per cell. Features are
matrix edits:

| Feature | Matrix change |
|---|---|
| Conference | every leg is a sink fed by all other legs (N-1) |
| Mute (leg X) | zero X's row (as a source), after the tap point |
| Deafen (leg X) | zero X's column (as a sink); RTP keeps flowing with silence/comfort noise |
| Supervisor listen | supervisor is a sink only |
| Whisper | supervisor is a source to the agent's sink only |
| Barge | supervisor is a source to all sinks and a sink of all |
| Hold | zero the held leg's source row; feed the shared hold source into its sink |
| Prompts | temporary source into selected sinks |

To bound cost, only the top 3-4 active speakers are mixed into each sink;
non-speaking sinks may share a single mix.

### 4.5 Compile-time limit

`pub const MAX_LEGS_PER_BRIDGE: usize = 32;` is defined once in `crates/core`.
The matrix and mixer derive their sizes from it (fixed-size arrays / const
generics), so changing the limit is a one-line edit and rebuild. Node-level
limits (total legs and bridges, STT concurrency) are runtime configuration
because they depend on the host.

### 4.6 Boundaries and threading

- Core to edge: a small trait (push/pull frame style) with a fake
  implementation for tests.
- Core to outside: commands in over a channel; events out. The gRPC layer
  attaches here.
- STT and intercept consumers sit behind bounded, droppable queues so a slow
  consumer can never delay the tick.
- The tick loop runs on a dedicated thread with no locks held while mixing.
  Async (tokio) is used for the control API and edge I/O only.

## 5. Control API (gRPC via tonic)

Resources and operations:

- Legs: `CreateLeg`, `UpdateLeg`, `ApplyAnswer`, `DeleteLeg`
- Bridges: `CreateBridge`; attach/detach a leg
- Matrix: `SetRoute(source, sink, gain)`, plus convenience calls `Mute`,
  `Deafen`, `Monitor` (listen/whisper/barge), `Hold`/`Resume`
- Playback: `Play(source, targets, loop)`, named shared hold sources
- Events (server stream): leg state, active speaker, DTMF, transcript
  segments, playback finished, capacity warnings
- Admin: health, drain mode, Prometheus metrics

## 6. Leg setup contract

`CreateLeg(requirements)` returns `{leg_id, sdp, negotiated}`.

Request (structured requirements, not SDP):

- Transport: RTP, SRTP (keying mode), or WebRTC (ICE/DTLS roles)
- Codec policy: ordered acceptable codecs, ptime, DTMF handling (RFC 4733)
- Direction: sendrecv, sendonly, recvonly
- Optional `remote_offer`: present for inbound calls (Nimossini returns an
  answer); absent for outbound calls (Nimossini returns an offer)

Response:

- `sdp`: the finished SDP body, ready to place in the SIP message
- `negotiated`: structured summary (chosen codec, payload types, SRTP/ICE
  state) so the control plane never parses SDP

Follow-ups: `ApplyAnswer(leg_id, remote_answer)` for the outbound case;
`UpdateLeg(leg_id, requirements, remote_offer?)` for re-INVITE and mid-call
codec changes, returning a new `sdp`.

SDP generation and parsing live entirely inside Nimossini. The same request
shape serves SIP and WebRTC legs.

## 7. Crate layout (Cargo workspace)

```
nimossini/
  crates/
    core/      pure Rust: matrix, mixer, gates, tick, taps (no GStreamer/tokio)
    edge/      GStreamer legs: RTP, SRTP, WebRTC, codecs
    proto/     protobuf definitions and generated types
    server/    binary: wires core + edge + gRPC + metrics
    stt/       STT worker interface (separate process, local model)
    ctl-poc/   minimal control plane: SIP UAS/B2BUA driving the server
```

Build order follows learning order: `core`, then `edge`, then `proto` and
`server`, then `ctl-poc`.

## 8. Failure handling

- Leg failure: edge reports; core drops the leg's row and column; event to the
  control plane. Other legs are unaffected (one pipeline per leg).
- Missing/late frames: substitute silence/concealment; the tick never waits.
- Slow consumers (STT, intercept delivery, event stream): bounded
  drop-oldest queues with counters. Intercept drops raise an alert but never
  affect the call.
- Tick overruns: measured and counted; sustained overruns put the node in a
  no-new-legs state and emit a capacity event.
- Control-plane disconnect: calls continue; after a configurable orphan
  timeout the server tears down uncontrolled legs.
- Process crash: per-call state is lost by design; the control plane decides.
- Rust errors: typed errors (`thiserror`) in `core` and `edge`; `anyhow` in
  `server`; no `unwrap` in the tick path.

## 9. Observability

- Prometheus metrics: legs/bridges, tick duration histogram and overruns,
  per-leg jitter/loss/packets, queue drops, STT latency, CPU headroom.
- Structured logs (`tracing`) with `leg_id`/`bridge_id` spans. Intercept
  activity is excluded from ordinary logs.
- Per-leg stats available over the API.

## 10. Capacity and admission control

- Per-leg cost model: decode + resample + VAD + encode + SRTP + mixing share,
  plus a weight for STT while transcribing. Numbers come from soak tests, not
  estimates.
- `CreateLeg` estimates the new leg's cost against a configured budget and
  returns `RESOURCE_EXHAUSTED` when over.
- Live signals (tick p99, CPU headroom) override the model and block new legs.
- STT has its own concurrency limit and degrades first (queue, then drop),
  never audio.
- Drain mode: refuse new legs, let current ones finish, report idle.

## 11. Legal intercept (design proposal)

Requirements depend on jurisdiction and regulatory status. Counsel must
confirm them before this is built.

- Separate listener and credentials (mTLS), with an authorization model
  distinct from the normal API. Operations: start for a target, stop, list.
- Tap points: T0 (encoded ingress, exactly what was on the wire) and T1
  (decoded, pre-gate). Both are before mute, deafen and hold, so no
  user-visible control can disable them.
- Per-direction streams, with call-identifying metadata on a separate channel.
- Delivery to a configured mediation function over an encrypted channel via a
  bounded queue. Stalls raise an alert and never affect the call.
- Invisibility: intercepts never appear in SDP, the normal API, ordinary
  events, metrics labels or normal logs. Only a dedicated audit log records
  start/stop, time and target.
- Code isolation: a dedicated module with a narrow interface into the core.
- PoC scope: prove tap points and isolation properties only.

## 12. Testing

1. `core` unit tests: synthetic tones/silence; mute, deafen, N-1, whisper,
   barge, top-K, tap ordering. Property tests (`proptest`): no overflow on
   clipping, N-1 never includes self.
2. Edge tests: loopback RTP between pipelines; compare decoded audio per codec
   pair, including a mid-call codec swap.
3. Server integration: gRPC client creates legs/bridges and verifies events and
   metrics; SIP/RTP test client (e.g. SIPp) for real call flows.
4. Soak/capacity tests: N synthetic calls and conferences; measure tick
   duration and CPU.
5. Intercept suite: proves taps survive mute, deafen, hold and consumer
   failure.

## 13. Learning-oriented delivery

- The implementation plan starts with warm-up milestones (ownership, traits,
  error handling, testing) before any GStreamer work.
- Each milestone states its goal, the Rust concepts it exercises, and tests
  written first.
- Hard topics (GStreamer pad probes, async plus threads, FFI) get isolated
  early spikes.
- The assistant's role is planning, explanation and review; the author writes
  the implementation.

## 14. Open questions

- Internal sample rate: 16 kHz proposed; confirm against the Opus path and the
  chosen STT engine.
- Rust SIP library choice (spike required; fallback documented in section 3).
- Local STT engine and model.
- Initial codec set (suggested: G.711 u-law/a-law, G.722, Opus; G.729 later).
- Toolchain setup: Rust and GStreamer are not yet installed on the dev
  machine.
