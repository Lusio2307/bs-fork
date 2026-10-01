# AGENTS.md

Guidance for AI agents (and humans) working in this repository.

## What this project is

A **fork** of JRF63's WebRTC desktop streamer: a Windows desktop-streaming server written in Rust,
in the same space as Steam Link, Parsec and Moonlight, with touch/pen passthrough that those don't
have.

**The goal of this fork:** a cleaner, more up-to-date project where I can stream my own Windows
desktop into a web app in the browser, keeping the original's focus on **low latency and high
performance**.

Pipeline, for orientation:

```
desktop ── DXGI Output Duplication ──▶ NVENC (H.264) ──▶ RTP ──▶ webrtc-rs ──▶ browser
                                                                                │
browser ── WebRTC data channel ──▶ synthetic pointer input injection ◀──────────┘
```

**Latency is the feature.** The original's stated design target is ~50 ms round trip (~16 ms in the
encoder, well under 1 ms on the wire, the remainder browser decode). Treat anything that adds
buffering, queueing or per-frame allocation to that path as a regression unless it is measured and
justified.

## Priorities, in order

1. It streams, reliably, at low latency.
2. It stays buildable and runnable at every step.
3. It gets cleaner and more current — **incrementally**.

When priorities conflict, the lower number wins.

## Ground rules

- **Step by step. No big refactorings.** Do not restructure modules, rename types, reorganise files,
  introduce new architectural layers, or reformat whole files. Keep diffs small and reviewable, and
  make one coherent change at a time.
- **Never mix concerns.** A dependency bump, a bug fix and a feature belong in three separate
  changes. This applies especially to dependency work.
- **Keep it working.** After every change the tree must still compile and still stream. Do not leave
  the repository in a half-migrated state, even on a branch.
- **Prefer deletion over addition.** Removing dead code and stale copies is a legitimate, valuable
  change; adding abstraction is not, unless the task actually requires it.
- **Don't reformat opportunistically.** An unrelated `cargo fmt` over a file you touched obscures
  the real diff. Format only what you change.
- **Ground claims in evidence.** Latency and performance claims need a measurement or a clear
  mechanism, not an assumption. Say so when you have not verified something.
- **Ask before widening scope.** If a fix turns out to require a large rewrite, stop and report
  rather than absorbing it into the current task.

## Updating outdated crates and components

This fork inherits dependencies that are years behind. Known drift (as reviewed):

| Crate | Locked | Latest at review |
| --- | --- | --- |
| `webrtc` | 0.6.0 | 0.21.0 |
| `windows` | 0.43.0 | 0.62.2 |
| `base64` | 0.13 | 0.22 line |
| `env_logger` | 0.10 | 0.11 line |
| `rand` | 0.8 | 0.9 line |
| `tokio` | 1.25 | 1.x (minor bumps) |
| `warp` | 0.3.3 | see upstream; `axum` is the mainstream alternative |

Rules for modernising:

- **One crate, or one coherent group, per change.** Never upgrade the whole tree at once, and never
  run a blind repo-wide `cargo update`.
- **Read the upstream changelog or migration guide first** for any major bump, and check that the
  APIs this project actually uses still exist.
- **Upgrade smallest and most independent first.** `windows` is mechanical and self-contained; do it
  before anything large.
- **Treat `webrtc` 0.6 → 0.21 as its own project.** The crate has been rearchitected (Sans-IO) since
  this pin, so expect substantial API churn across `webrtc-helper`. Do not attempt it as a drive-by
  bump, and do not start it before CI exists and the ordinary fixes are done.
- **Verify after each step:** `cargo check --workspace --all-targets`, then an end-to-end session in
  a browser. Keep `Cargo.lock` updated and committed.
- **If a bump forces wide-reaching edits, stop** and split it into a dedicated change rather than
  letting it sprawl through the codebase.

## Upstream reference

The `upstream` remote points at https://github.com/JRF63/desktop-streaming. Two branches matter:

- **`upstream/old`** — the branch this fork is based on. Its tip is this fork's original HEAD
  (`c8622c1`), so there is nothing to pull from it.
- **`upstream/dev`** — 33 commits made after this fork point (2023-10 to 2024-01), then abandoned
  mid-restructure, which is why `old` is still the default branch. It is a crates-only
  decomposition (`audio-codec`, `audio-source`, `conveyor-buffer`, `inputs`, `video-source`,
  `webrtc-bridge`, `windows-util`) and **contains no server binary, no encoder and no web client**,
  so it is **not a usable base** for this fork and must not be merged wholesale.

Treat `upstream/dev` as a reference and a parts bin, not a merge target. It is useful for:

- its Opus and loopback-audio implementation (`audio-codec`, `audio-source`) — the audio feature
  this fork still lacks;
- worked examples of the `windows` 0.43 → 0.52 and `webrtc` 0.6 → 0.9 bumps;
- corroboration of this fork's own direction: upstream likewise moved `webrtc-helper` into the
  repository (renamed `webrtc-bridge`), set `resolver = "2"`, and put `license` in the workspace
  manifest.

Port from it selectively, one concern per change.

## Latency-critical code — handle with care

- `server-windows/src/nvidia/encoder.rs` — the capture/encode/pacing loop, RTP timestamp
  accumulation, VBV buffer sizing, and bitrate updates driven by TWCC. The frame interval and the
  VBV buffer size are the two knobs that most directly affect latency.
- `webrtc-helper/src/codecs/h264/` — NAL fragmentation and payloading. Avoid adding allocation or
  copying per frame.
- `webrtc-helper/src/encoder/track.rs` — the hand-off between the peer connection and the encoder.

In this code: prefer dropping frames over queueing them, keep hot-path channels bounded and cheap,
avoid blocking or `block_on` inside async tasks, and never introduce a queue whose depth grows with
network jitter.

## Repository layout

| Path | What it is |
| --- | --- |
| `server-windows/` | The Windows server: capture, NVENC encoder, warp HTTP/WebSocket server, HTML client |
| `webrtc-helper/` | Wrapper over `webrtc-rs` exposing `EncoderBuilder` / `DecoderBuilder` / `Signaler` traits |
| `nvenc-rs/` | Vendored upstream providing the `nvenc` / `nvenc-sys` crates |
| `ENHANCEMENT-PLAN.md` | Phased improvement plan (git-ignored in this clone) |

The workspace root `Cargo.toml` includes `server-windows` and `webrtc-helper`, and excludes
`nvenc-rs`, which forms a workspace of its own.

Vendored upstreams: `webrtc-helper/` and `nvenc-rs/` are ordinary directories of this repository,
not git submodules, and the unused `client-android` has been dropped. Do not reintroduce them as
submodules. Both keep their own LICENSE files and copyright notices, and their provenance,
including the pins they were taken from, is recorded in `NOTICE`.

## Prerequisites

- Windows 10/11 (the server is Windows-only; DXGI and NVENC)
- An **NVIDIA GPU** with a recent driver — H.264 via NVENC is the only encoder implemented today
- MSVC build toolchain (`rustup default stable-msvc`)
- **libclang** on `PATH` or via `LIBCLANG_PATH` — `nvenc-sys` runs `bindgen` at build time
- Git (there are no submodules to initialise — the upstreams are vendored)

## Build, run and verify

```sh
cargo check --workspace --all-targets   # fast feedback
cargo clippy --workspace --all-targets
cargo test  --workspace                 # some tests need a GPU
cargo run --release                     # then browse to http://<pc-ip>:9090
```

- Run from the repository root: debug builds read the client HTML from a hardcoded relative path.
- Set `RUST_LOG=info` to see the server's log output.
- Some tests (`capture.rs`, `device.rs`) require real GPU hardware and will fail on machines without
  it — that is expected, not a regression.
- There is **no CI** yet. Verification is manual: build, run, connect from a browser, confirm video
  and input.

**Definition of done for a change:** it builds, it still streams end to end, input still works, the
diff is small and does one thing, and any latency or performance claim is measured or explained.

## Known rough edges

These are known and being addressed incrementally — don't "fix" them by accident as part of
unrelated work, and don't be surprised by them:

- **Single client at a time**, enforced by a flag in `server-windows/src/server.rs`.
- **No authentication and no TLS.** The server binds all interfaces and grants full remote desktop
  control to anyone who can reach the port. Never expose it to an untrusted network.
- **No keyboard or mouse-wheel input yet** — only pointer, touch and pen.
- **H.264 only.** HEVC and AV1 paths exist as `todo!()` stubs.
- **One failed session can wedge the server** until restarted, and a disconnect can spin a CPU core.
- **Frame pacing is hardcoded to 60 fps**, independent of the display's refresh rate.
- Resolution changes are not handled; width and height are fixed when the encoder is built.

## Reference

- `ENHANCEMENT-PLAN.md` — the ordered, phased plan for the work above, with file and line
  references and acceptance criteria. Git-ignored in this clone (see `.git/info/exclude`), so it
  will not appear in `git status`.
- `README.md` — the upstream project's own description, usage and TODO list.
