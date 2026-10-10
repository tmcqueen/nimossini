# Nimossini Core Crate Implementation Plan (Plan 1 of N)

> **How this plan is executed:** the author implements it personally to learn
> Rust. The assistant plans, explains and reviews. Tasks give the full failing
> tests, exact signatures and hints; the implementations are deliberately left
> to the author. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Build `nimossini-core`: a pure-Rust mixing core (frames, gain, mixing
matrix, gates, level/top-K selection, taps, sources, and a 20 ms `Bridge::tick`)
fully covered by tests, with no GStreamer or tokio dependency.

**Architecture:** One `Frame` (mono i16 PCM, 20 ms) flows through a deterministic
`tick`: levels, taps (pre-gate), mute, top-K selection, per-sink matrix mixing,
deafen. Everything is plain data plus small traits, so it is testable with
synthetic audio.

**Tech Stack:** Rust (stable), cargo workspace, `thiserror`, `proptest`
(dev), `hound` (dev, example only).

**Spec:** `docs/superpowers/specs/2026-10-10-nimossini-media-plane-design.md`
(sections 4, 8, 12).

## Global Constraints

- Frames are mono, 16-bit PCM, 20 ms; proposed internal rate 16 kHz, so
  `FRAME_SAMPLES = 320`.
- `MAX_LEGS_PER_BRIDGE: usize = 32` is defined once, in `crates/core`; all
  matrix and mixer sizes derive from it.
- `crates/core` has no GStreamer and no tokio dependency.
- Typed errors with `thiserror` in `core`; no `unwrap`/`expect`/indexing panics
  in the tick path (tests may unwrap).
- Taps are called before any gate (mute/deafen), and a slow tap never blocks the
  tick.
- Self-routing is impossible: a sink never hears its own source (N-1).
- Code must pass `cargo fmt --check` and `cargo clippy -- -D warnings`.

## Review Focus

Failure modes the spec implies; each has a pinning test in the owning task:

1. Wrong-length audio buffers (empty, short, long) return an error and never
   panic. Pinned in Task 1.
2. Many loud sources summed past the i16 range clamp, with no wrap-around and no
   debug-build overflow panic. Pinned in Task 2.
3. Non-finite or negative gains (NaN, infinity, -0.1) are rejected. Pinned in
   Task 2.
4. A late packet for a slot that is no longer attached is ignored, and produces
   no output. Pinned in Task 8.
5. A zero-capacity or overfull tap queue drops and counts, and never blocks or
   panics. Pinned in Task 7.

## File Structure

```
nimossini/
  Cargo.toml                      workspace root
  .gitignore
  crates/core/
    Cargo.toml
    src/
      lib.rs                      constants + module declarations
      error.rs                    CoreError
      frame.rs                    Frame
      gain.rs                     Gain
      slot.rs                     Slot (validated index)
      mixer.rs                    Accumulator, Inputs, mix_for_sink
      matrix.rs                   Matrix
      gates.rs                    Gates
      level.rs                    rms, dbfs, top_k
      queue.rs                    DropQueue
      tap.rs                      Tap trait, QueueTap, TapHandle
      source.rs                   Source trait, ToneSource
      bridge.rs                   Bridge, Outputs, tick
    tests/
      bridge.rs                   integration tests through the public API
      properties.rs               proptest invariants
    examples/
      three_party.rs              writes WAV files so you can listen
      tick_cost.rs                prints average tick time for 32 legs
```

Unit tests live inline in each module (`#[cfg(test)] mod tests`). Integration
tests go in `tests/`.

---

### Task 0: Toolchain and workspace

**Concepts:** rustup, cargo workspaces, `cargo new`, `cargo add`, fmt/clippy.

**Files:**
- Create: `Cargo.toml`, `.gitignore`, `crates/core/*` (via cargo)

**Produces:** a building, empty `nimossini-core` library in a workspace.

- [ ] **Step 1: Install Rust**

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustc --version && cargo --version
```
Expected: version numbers print. If a later build fails with a linker error,
install a C toolchain: `sudo apt install build-essential`.

- [ ] **Step 2: Create the workspace**

```bash
cd ~/nimossini
printf '[workspace]\nresolver = "2"\nmembers = ["crates/*"]\n' > Cargo.toml
printf '/target\n' > .gitignore
cargo new --lib crates/core --name nimossini-core
cargo add thiserror -p nimossini-core
cargo add --dev proptest hound -p nimossini-core
```

- [ ] **Step 3: Replace `crates/core/src/lib.rs`**

```rust
//! Nimossini core: the pure-Rust mixing engine.

/// Internal sample rate (Hz). To be confirmed in the edge milestone.
pub const SAMPLE_RATE_HZ: u32 = 16_000;
/// Frame duration in milliseconds.
pub const FRAME_MS: u32 = 20;
/// Samples per frame (mono).
pub const FRAME_SAMPLES: usize = 320;
/// Maximum legs (plus sources) in one bridge. Change here, rebuild.
pub const MAX_LEGS_PER_BRIDGE: usize = 32;
/// How many of the loudest speakers are mixed into each sink.
pub const MIX_TOP_K: usize = 4;
/// A slot must be louder than this to count as "speaking".
pub const SPEECH_THRESHOLD_DBFS: f32 = -45.0;

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn frame_constants_are_consistent() {
        assert_eq!(
            FRAME_SAMPLES,
            SAMPLE_RATE_HZ as usize * FRAME_MS as usize / 1000
        );
    }
}
```

- [ ] **Step 4: Verify**

```bash
cargo test && cargo fmt --check && cargo clippy -- -D warnings
```
Expected: 1 test passes; fmt and clippy are clean.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "chore: scaffold cargo workspace and core crate"
```

---

### Task 1: Errors and `Frame` (warm-up: ownership, `Result`, arrays)

**Concepts:** structs, `Copy` vs `Clone`, fixed-size arrays, `Result`, derive,
`thiserror`, inline unit tests.

**Files:**
- Create: `crates/core/src/error.rs`, `crates/core/src/frame.rs`
- Modify: `crates/core/src/lib.rs` (add `pub mod error; pub mod frame;`)

**Produces:**
```rust
// error.rs
#[derive(Debug, PartialEq, Eq, thiserror::Error)]
pub enum CoreError {
    BadFrameLength { expected: usize, got: usize },
    SlotOutOfRange(usize),
    SelfRoute(usize),
    InvalidGain,
}
// frame.rs
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct Frame([i16; FRAME_SAMPLES]);
impl Frame {
    pub fn silence() -> Self;
    pub fn filled(value: i16) -> Self;
    pub fn from_slice(data: &[i16]) -> Result<Self, CoreError>;
    pub fn samples(&self) -> &[i16; FRAME_SAMPLES];
    pub fn is_silence(&self) -> bool;
}
```

- [ ] **Step 1: Write the failing tests** (bottom of `frame.rs`)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn silence_is_all_zero() {
        let f = Frame::silence();
        assert!(f.is_silence());
        assert!(f.samples().iter().all(|&s| s == 0));
    }

    #[test]
    fn filled_sets_every_sample() {
        let f = Frame::filled(7);
        assert!(f.samples().iter().all(|&s| s == 7));
        assert!(!f.is_silence());
    }

    #[test]
    fn from_slice_accepts_exact_length() {
        let data = vec![1i16; FRAME_SAMPLES];
        assert_eq!(Frame::from_slice(&data).unwrap(), Frame::filled(1));
    }

    #[test]
    fn from_slice_rejects_wrong_length() {
        assert_eq!(
            Frame::from_slice(&[1, 2, 3]),
            Err(CoreError::BadFrameLength { expected: FRAME_SAMPLES, got: 3 })
        );
        assert!(Frame::from_slice(&[]).is_err());
        assert!(Frame::from_slice(&vec![0i16; FRAME_SAMPLES + 1]).is_err());
    }
}
```

- [ ] **Step 2: Run, expect FAIL** (does not compile yet)

Run: `cargo test -p nimossini-core frame`
Expected: errors about missing `Frame` / `CoreError`.

- [ ] **Step 3: Implement** `CoreError` (with `#[error("...")]` messages) and
  `Frame`. Hints: `[v; N]` repeats a value; `<[i16; N]>::try_from(slice)`
  converts a slice, or check `data.len()` yourself; `iter().all(...)`.

- [ ] **Step 4: Run, expect PASS**

Run: `cargo test -p nimossini-core && cargo clippy -- -D warnings`

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(core): add Frame and CoreError"
```

---

### Task 2: `Gain` and `Accumulator` (warm-up: newtypes, numeric safety)

**Concepts:** newtype pattern, validated constructors, associated consts, integer
vs float arithmetic, why clamp *once* at the end.

**Files:**
- Create: `crates/core/src/gain.rs`, `crates/core/src/mixer.rs`
- Modify: `lib.rs` (add `pub mod gain; pub mod mixer;`)

**Consumes:** `Frame`, `CoreError`, `FRAME_SAMPLES`, `MAX_LEGS_PER_BRIDGE`.

**Produces:**
```rust
// gain.rs
#[derive(Clone, Copy, Debug, PartialEq)]
pub struct Gain(f32);
impl Gain {
    pub const ZERO: Gain;
    pub const UNITY: Gain;
    pub fn new(v: f32) -> Result<Gain, CoreError>; // finite and >= 0.0
    pub fn value(self) -> f32;
}
// mixer.rs
pub struct Accumulator { /* [i32; FRAME_SAMPLES] */ }
impl Accumulator {
    pub fn new() -> Self;
    pub fn add(&mut self, frame: &Frame, gain: Gain);
    pub fn finish(&self) -> Frame; // clamps to i16 range
}
```

- [ ] **Step 1: Write the failing tests**

`gain.rs`:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn unity_and_zero() {
        assert_eq!(Gain::UNITY.value(), 1.0);
        assert_eq!(Gain::ZERO.value(), 0.0);
    }

    #[test]
    fn accepts_normal_values() {
        assert!(Gain::new(0.5).is_ok());
        assert!(Gain::new(2.0).is_ok());
    }

    #[test]
    fn rejects_bad_values() {
        for bad in [f32::NAN, f32::INFINITY, f32::NEG_INFINITY, -0.1] {
            assert_eq!(Gain::new(bad), Err(CoreError::InvalidGain));
        }
    }
}
```

`mixer.rs`:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::MAX_LEGS_PER_BRIDGE;

    #[test]
    fn adds_two_frames() {
        let mut acc = Accumulator::new();
        acc.add(&Frame::filled(100), Gain::UNITY);
        acc.add(&Frame::filled(200), Gain::UNITY);
        assert_eq!(acc.finish(), Frame::filled(300));
    }

    #[test]
    fn gain_scales() {
        let mut acc = Accumulator::new();
        acc.add(&Frame::filled(1000), Gain::new(0.5).unwrap());
        assert_eq!(acc.finish(), Frame::filled(500));
    }

    #[test]
    fn clips_instead_of_wrapping() {
        let mut acc = Accumulator::new();
        acc.add(&Frame::filled(30000), Gain::UNITY);
        acc.add(&Frame::filled(30000), Gain::UNITY);
        assert_eq!(acc.finish(), Frame::filled(i16::MAX));

        let mut acc = Accumulator::new();
        acc.add(&Frame::filled(-30000), Gain::UNITY);
        acc.add(&Frame::filled(-30000), Gain::UNITY);
        assert_eq!(acc.finish(), Frame::filled(i16::MIN));
    }

    #[test]
    fn many_loud_sources_do_not_overflow() {
        let mut acc = Accumulator::new();
        for _ in 0..MAX_LEGS_PER_BRIDGE {
            acc.add(&Frame::filled(i16::MAX), Gain::UNITY);
        }
        assert_eq!(acc.finish(), Frame::filled(i16::MAX));
    }
}
```

- [ ] **Step 2: Run, expect FAIL**

Run: `cargo test -p nimossini-core gain mixer`

- [ ] **Step 3: Implement.** Hints: accumulate in `i32`, scale each sample as
  `(s as f32 * gain) as i32`, clamp once in `finish` with `.clamp(i16::MIN as i32,
  i16::MAX as i32)`. `Gain::new` can use `v.is_finite() && v >= 0.0`.
  Why accumulate in `i32`: summing in `i16` would overflow (a panic in debug
  builds).

- [ ] **Step 4: Run, expect PASS** (`cargo test`, then clippy)

- [ ] **Step 5: Commit** — `git commit -am "feat(core): add Gain and Accumulator"`

---

### Task 3: `Slot` and `Matrix`

**Concepts:** const-sized arrays driven by a constant, validated indices,
returning `Result`, `&mut self`.

**Files:**
- Create: `crates/core/src/slot.rs`, `crates/core/src/matrix.rs`
- Modify: `lib.rs` (add `pub mod slot; pub mod matrix;`)

**Produces:**
```rust
// slot.rs
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct Slot(usize);
impl Slot {
    pub fn new(index: usize) -> Result<Slot, CoreError>; // < MAX_LEGS_PER_BRIDGE
    pub fn index(self) -> usize;
}
// matrix.rs
pub struct Matrix { /* [[Gain; MAX_LEGS_PER_BRIDGE]; MAX_LEGS_PER_BRIDGE] */ }
impl Matrix {
    pub fn new() -> Self; // all ZERO
    pub fn set(&mut self, source: Slot, sink: Slot, gain: Gain) -> Result<(), CoreError>;
    pub fn get(&self, source: Slot, sink: Slot) -> Gain;
    pub fn connect_all(&mut self, slots: &[Slot]); // full N-1 mesh at UNITY
}
```
`set` with `source == sink` returns `Err(CoreError::SelfRoute(index))`.

- [ ] **Step 1: Write the failing tests**

`slot.rs`:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_in_range() {
        assert!(Slot::new(0).is_ok());
        assert!(Slot::new(MAX_LEGS_PER_BRIDGE - 1).is_ok());
    }

    #[test]
    fn rejects_out_of_range() {
        assert_eq!(
            Slot::new(MAX_LEGS_PER_BRIDGE),
            Err(CoreError::SlotOutOfRange(MAX_LEGS_PER_BRIDGE))
        );
    }
}
```

`matrix.rs`:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn s(i: usize) -> Slot {
        Slot::new(i).unwrap()
    }

    #[test]
    fn new_matrix_is_all_zero() {
        let m = Matrix::new();
        for a in 0..MAX_LEGS_PER_BRIDGE {
            for b in 0..MAX_LEGS_PER_BRIDGE {
                assert_eq!(m.get(s(a), s(b)), Gain::ZERO);
            }
        }
    }

    #[test]
    fn set_then_get() {
        let mut m = Matrix::new();
        m.set(s(1), s(2), Gain::UNITY).unwrap();
        assert_eq!(m.get(s(1), s(2)), Gain::UNITY);
        assert_eq!(m.get(s(2), s(1)), Gain::ZERO); // direction matters
    }

    #[test]
    fn self_route_is_rejected() {
        let mut m = Matrix::new();
        assert_eq!(m.set(s(2), s(2), Gain::UNITY), Err(CoreError::SelfRoute(2)));
    }

    #[test]
    fn connect_all_is_a_mesh_without_self_routes() {
        let mut m = Matrix::new();
        m.connect_all(&[s(0), s(1), s(2)]);
        assert_eq!(m.get(s(0), s(1)), Gain::UNITY);
        assert_eq!(m.get(s(2), s(0)), Gain::UNITY);
        assert_eq!(m.get(s(1), s(1)), Gain::ZERO);
        assert_eq!(m.get(s(0), s(3)), Gain::ZERO); // not in the set
    }
}
```

- [ ] **Step 2: Run, expect FAIL** — `cargo test -p nimossini-core slot matrix`
- [ ] **Step 3: Implement.** Hint: `[[Gain::ZERO; N]; N]` works because `Gain` is
  `Copy`. Try changing `MAX_LEGS_PER_BRIDGE` to 8, run the tests, and put it
  back: nothing else should need editing.
- [ ] **Step 4: Run, expect PASS** (tests, fmt, clippy)
- [ ] **Step 5: Commit** — `git commit -am "feat(core): add Slot and Matrix"`

---

### Task 4: `mix_for_sink`

**Concepts:** slices and arrays of `Option<T>`, iterating with indices,
`Option` handling without `unwrap`.

**Files:** Modify `crates/core/src/mixer.rs`

**Consumes:** `Matrix::get`, `Accumulator`, `Slot`.

**Produces:**
```rust
pub type Inputs = [Option<Frame>; MAX_LEGS_PER_BRIDGE];
pub fn mix_for_sink(matrix: &Matrix, inputs: &Inputs, sink: Slot) -> Frame;
```
It sums `inputs[source]` weighted by `matrix.get(source, sink)` for every source.

- [ ] **Step 1: Write the failing tests** (append to `mixer.rs` tests; add `use
  crate::{matrix::Matrix, slot::Slot};`)

```rust
    fn s(i: usize) -> Slot {
        Slot::new(i).unwrap()
    }

    fn inputs_with(pairs: &[(usize, Frame)]) -> Inputs {
        let mut v: Inputs = [None; MAX_LEGS_PER_BRIDGE];
        for (i, f) in pairs {
            v[*i] = Some(*f);
        }
        v
    }

    #[test]
    fn n_minus_one_excludes_self() {
        let mut m = Matrix::new();
        m.connect_all(&[s(0), s(1), s(2)]);
        let inputs = inputs_with(&[
            (0, Frame::filled(100)),
            (1, Frame::filled(200)),
            (2, Frame::filled(400)),
        ]);
        assert_eq!(mix_for_sink(&m, &inputs, s(0)), Frame::filled(600));
        assert_eq!(mix_for_sink(&m, &inputs, s(1)), Frame::filled(500));
        assert_eq!(mix_for_sink(&m, &inputs, s(2)), Frame::filled(300));
    }

    #[test]
    fn a_one_way_route_only_reaches_its_sink() {
        let mut m = Matrix::new();
        m.set(s(3), s(0), Gain::UNITY).unwrap();
        let inputs = inputs_with(&[(3, Frame::filled(900))]);
        assert_eq!(mix_for_sink(&m, &inputs, s(0)), Frame::filled(900));
        assert!(mix_for_sink(&m, &inputs, s(1)).is_silence());
    }

    #[test]
    fn missing_inputs_are_skipped() {
        let mut m = Matrix::new();
        m.connect_all(&[s(0), s(1), s(2)]);
        let inputs = inputs_with(&[(1, Frame::filled(200))]);
        assert_eq!(mix_for_sink(&m, &inputs, s(0)), Frame::filled(200));
    }

    #[test]
    fn no_inputs_gives_silence() {
        let m = Matrix::new();
        let inputs: Inputs = [None; MAX_LEGS_PER_BRIDGE];
        assert!(mix_for_sink(&m, &inputs, s(0)).is_silence());
    }
```

- [ ] **Step 2: Run, expect FAIL** — `cargo test -p nimossini-core mixer`
- [ ] **Step 3: Implement.** Hint: `for (i, input) in inputs.iter().enumerate()`,
  `if let Some(frame) = input`, skip when the gain is `Gain::ZERO` (cheap
  optimization, and it must not change results).
- [ ] **Step 4: Run, expect PASS**
- [ ] **Step 5: Commit** — `git commit -am "feat(core): mix per-sink from matrix"`

---

### Task 5: `Gates`

**Concepts:** small state types, bool arrays, getters/setters.

**Files:** Create `crates/core/src/gates.rs`; modify `lib.rs` (`pub mod gates;`)

**Produces:**
```rust
pub struct Gates { /* muted, deafened: [bool; MAX_LEGS_PER_BRIDGE] */ }
impl Gates {
    pub fn new() -> Self; // all open
    pub fn set_muted(&mut self, slot: Slot, muted: bool);
    pub fn set_deafened(&mut self, slot: Slot, deafened: bool);
    pub fn is_muted(&self, slot: Slot) -> bool;
    pub fn is_deafened(&self, slot: Slot) -> bool;
}
```

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn s(i: usize) -> Slot {
        Slot::new(i).unwrap()
    }

    #[test]
    fn everything_starts_open() {
        let g = Gates::new();
        assert!(!g.is_muted(s(0)));
        assert!(!g.is_deafened(s(0)));
    }

    #[test]
    fn mute_and_deafen_are_independent() {
        let mut g = Gates::new();
        g.set_muted(s(1), true);
        assert!(g.is_muted(s(1)));
        assert!(!g.is_deafened(s(1)));
        assert!(!g.is_muted(s(2)));
    }

    #[test]
    fn gates_can_be_cleared() {
        let mut g = Gates::new();
        g.set_deafened(s(3), true);
        g.set_deafened(s(3), false);
        assert!(!g.is_deafened(s(3)));
    }
}
```

- [ ] **Step 2: Run, expect FAIL** — `cargo test -p nimossini-core gates`
- [ ] **Step 3: Implement.** Hint: `slot.index()` indexes the arrays safely
  because `Slot` is validated at construction.
- [ ] **Step 4: Run, expect PASS**
- [ ] **Step 5: Commit** — `git commit -am "feat(core): add mute and deafen gates"`

---

### Task 6: Levels and top-K

**Concepts:** `f32` maths, iterators, sorting with `partial_cmp`, `Option`.

**Files:** Create `crates/core/src/level.rs`; modify `lib.rs` (`pub mod level;`)

**Produces:**
```rust
pub const DBFS_FLOOR: f32 = -96.0;
pub fn rms(frame: &Frame) -> f32;   // 0.0 ..= 32768.0
pub fn dbfs(frame: &Frame) -> f32;  // 20*log10(rms/32768), floored at DBFS_FLOOR
pub type Levels = [Option<f32>; MAX_LEGS_PER_BRIDGE]; // dBFS per slot
pub type Selected = [bool; MAX_LEGS_PER_BRIDGE];
/// Slots louder than SPEECH_THRESHOLD_DBFS, loudest `k` of them.
pub fn top_k(levels: &Levels, k: usize) -> Selected;
```

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn silence_has_zero_rms_and_floor_dbfs() {
        assert_eq!(rms(&Frame::silence()), 0.0);
        assert_eq!(dbfs(&Frame::silence()), DBFS_FLOOR);
    }

    #[test]
    fn constant_frame_rms_equals_its_value() {
        assert!((rms(&Frame::filled(1000)) - 1000.0).abs() < 1e-3);
    }

    #[test]
    fn full_scale_is_near_zero_dbfs() {
        assert!(dbfs(&Frame::filled(i16::MAX)).abs() < 0.01);
    }

    fn levels_with(pairs: &[(usize, f32)]) -> Levels {
        let mut l: Levels = [None; MAX_LEGS_PER_BRIDGE];
        for (i, v) in pairs {
            l[*i] = Some(*v);
        }
        l
    }

    #[test]
    fn picks_the_loudest_k() {
        let l = levels_with(&[(0, -30.0), (1, -10.0), (2, -20.0), (3, -40.0)]);
        let sel = top_k(&l, 2);
        assert!(sel[1] && sel[2]);
        assert!(!sel[0] && !sel[3]);
    }

    #[test]
    fn quiet_slots_are_never_selected() {
        let l = levels_with(&[(0, -60.0), (1, -20.0)]);
        let sel = top_k(&l, 4);
        assert!(sel[1]);
        assert!(!sel[0]);
    }

    #[test]
    fn absent_slots_are_never_selected() {
        let sel = top_k(&[None; MAX_LEGS_PER_BRIDGE], 4);
        assert!(sel.iter().all(|&b| !b));
    }

    #[test]
    fn k_zero_selects_nothing() {
        let l = levels_with(&[(0, -10.0)]);
        assert!(top_k(&l, 0).iter().all(|&b| !b));
    }
}
```
(Imports needed in the module: `use crate::{frame::Frame, MAX_LEGS_PER_BRIDGE,
SPEECH_THRESHOLD_DBFS};`)

- [ ] **Step 2: Run, expect FAIL** — `cargo test -p nimossini-core level`
- [ ] **Step 3: Implement.** Hints: RMS = `sqrt(mean(s*s))` computed in `f32`/`f64`;
  `f32::log10`; collect `(index, level)` pairs above the threshold into a
  `Vec`, sort descending with `b.1.partial_cmp(&a.1)` (floats are not `Ord`;
  ask yourself why), take `k`.
- [ ] **Step 4: Run, expect PASS**
- [ ] **Step 5: Commit** — `git commit -am "feat(core): add level metering and top-K"`

---

### Task 7: `DropQueue` and taps (Review Focus 5)

**Concepts:** `VecDeque`, trait objects, `Arc<Mutex<_>>`, `try_lock` to avoid
blocking, atomics.

**Files:** Create `crates/core/src/queue.rs`, `crates/core/src/tap.rs`; modify
`lib.rs` (`pub mod queue; pub mod tap;`)

**Produces:**
```rust
// queue.rs
pub struct DropQueue<T> { /* VecDeque<T>, capacity, dropped: u64 */ }
impl<T> DropQueue<T> {
    pub fn new(capacity: usize) -> Self;
    pub fn push(&mut self, item: T);      // drops the oldest when full
    pub fn pop(&mut self) -> Option<T>;
    pub fn len(&self) -> usize;
    pub fn dropped(&self) -> u64;
}
// tap.rs
pub trait Tap: Send {
    fn on_frame(&mut self, slot: Slot, frame: &Frame);
}
pub struct QueueTap { /* shared queue + contention counter */ }
pub struct TapHandle { /* same shared state */ }
impl TapHandle {
    pub fn pop(&self) -> Option<(Slot, Frame)>;
    pub fn dropped(&self) -> u64; // queue drops + contended drops
}
pub fn queue_tap(capacity: usize) -> (QueueTap, TapHandle);
// QueueTap: impl Tap. It must use try_lock: if the consumer holds the lock the
// frame is dropped and counted; the tick never waits.
```

- [ ] **Step 1: Write the failing tests**

`queue.rs`:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn is_fifo() {
        let mut q = DropQueue::new(3);
        q.push(1);
        q.push(2);
        assert_eq!(q.pop(), Some(1));
        assert_eq!(q.pop(), Some(2));
        assert_eq!(q.pop(), None);
    }

    #[test]
    fn drops_the_oldest_when_full() {
        let mut q = DropQueue::new(2);
        q.push(1);
        q.push(2);
        q.push(3);
        assert_eq!(q.dropped(), 1);
        assert_eq!(q.pop(), Some(2));
        assert_eq!(q.pop(), Some(3));
    }

    #[test]
    fn zero_capacity_drops_everything_without_panicking() {
        let mut q = DropQueue::new(0);
        q.push(1);
        assert_eq!(q.len(), 0);
        assert_eq!(q.dropped(), 1);
        assert_eq!(q.pop(), None);
    }
}
```

`tap.rs`:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn s(i: usize) -> Slot {
        Slot::new(i).unwrap()
    }

    #[test]
    fn handle_receives_frames_in_order() {
        let (mut tap, handle) = queue_tap(8);
        tap.on_frame(s(1), &Frame::filled(10));
        tap.on_frame(s(2), &Frame::filled(20));
        assert_eq!(handle.pop(), Some((s(1), Frame::filled(10))));
        assert_eq!(handle.pop(), Some((s(2), Frame::filled(20))));
        assert_eq!(handle.pop(), None);
    }

    #[test]
    fn overflow_drops_and_counts() {
        let (mut tap, handle) = queue_tap(1);
        tap.on_frame(s(0), &Frame::filled(1));
        tap.on_frame(s(0), &Frame::filled(2));
        assert_eq!(handle.dropped(), 1);
        assert_eq!(handle.pop(), Some((s(0), Frame::filled(2))));
    }

    #[test]
    fn zero_capacity_tap_never_panics() {
        let (mut tap, handle) = queue_tap(0);
        tap.on_frame(s(0), &Frame::filled(1));
        assert_eq!(handle.pop(), None);
        assert_eq!(handle.dropped(), 1);
    }
}
```

- [ ] **Step 2: Run, expect FAIL** — `cargo test -p nimossini-core queue tap`
- [ ] **Step 3: Implement.** Hints: `VecDeque::pop_front`/`push_back`; for the tap,
  `Arc<Mutex<DropQueue<(Slot, Frame)>>>` plus an `Arc<AtomicU64>` for contended
  drops; `if let Ok(mut q) = self.shared.try_lock()`. Discuss with the
  assistant: why not `lock()`?
- [ ] **Step 4: Run, expect PASS**
- [ ] **Step 5: Commit** — `git commit -am "feat(core): add drop queue and queue tap"`

---

### Task 8: `Source` and `Bridge::tick` (Review Focus 4)

**Concepts:** trait objects (`Box<dyn Trait>`), composing modules, `Option`
combinators, integration tests in `tests/`.

**Files:**
- Create: `crates/core/src/source.rs`, `crates/core/src/bridge.rs`,
  `crates/core/tests/bridge.rs`
- Modify: `lib.rs` (`pub mod source; pub mod bridge;`)

**Consumes:** everything from Tasks 1-7.

**Produces:**
```rust
// source.rs
pub trait Source: Send {
    fn next_frame(&mut self) -> Option<Frame>;
}
// bridge.rs
pub type Outputs = [Option<Frame>; MAX_LEGS_PER_BRIDGE];
pub struct Bridge { /* matrix, gates, attached, pinned, sources, taps */ }
impl Bridge {
    pub fn new() -> Self;
    pub fn attach(&mut self, slot: Slot);                 // a leg
    pub fn detach(&mut self, slot: Slot);
    pub fn attach_source(&mut self, slot: Slot, src: Box<dyn Source>); // pinned
    pub fn set_pinned(&mut self, slot: Slot, pinned: bool);
    pub fn matrix_mut(&mut self) -> &mut Matrix;
    pub fn gates_mut(&mut self) -> &mut Gates;
    pub fn add_tap(&mut self, tap: Box<dyn Tap>);
    pub fn tick(&mut self, inputs: &Inputs) -> Outputs;
}
```

`tick` semantics (from the spec, section 4.3):
1. Build effective inputs: attached legs use `inputs[slot]`; source slots use
   `source.next_frame()`; **everything else is ignored**.
2. Compute dBFS per attached leg that has a frame.
3. Call every tap for each attached leg with a frame (pre-gate).
4. Muted legs are removed from the effective inputs.
5. Select `top_k(levels, MIX_TOP_K)`. A non-pinned leg that is not selected is
   removed; pinned slots (sources, supervisors) always stay.
6. For each attached sink, output `mix_for_sink`; a deafened sink outputs
   `Some(Frame::silence())`. Unattached slots output `None`.

- [ ] **Step 1: Write the failing tests** (`crates/core/tests/bridge.rs`)

```rust
use nimossini_core::bridge::{Bridge, Outputs};
use nimossini_core::frame::Frame;
use nimossini_core::gain::Gain;
use nimossini_core::mixer::Inputs;
use nimossini_core::slot::Slot;
use nimossini_core::source::Source;
use nimossini_core::tap::queue_tap;
use nimossini_core::{MAX_LEGS_PER_BRIDGE, MIX_TOP_K};

fn s(i: usize) -> Slot {
    Slot::new(i).unwrap()
}

fn inputs_with(pairs: &[(usize, i16)]) -> Inputs {
    let mut v: Inputs = [None; MAX_LEGS_PER_BRIDGE];
    for (i, val) in pairs {
        v[*i] = Some(Frame::filled(*val));
    }
    v
}

fn conference(slots: &[usize]) -> Bridge {
    let mut b = Bridge::new();
    let ss: Vec<Slot> = slots.iter().map(|&i| s(i)).collect();
    for &sl in &ss {
        b.attach(sl);
    }
    b.matrix_mut().connect_all(&ss);
    b
}

struct ConstSource(Frame);
impl Source for ConstSource {
    fn next_frame(&mut self) -> Option<Frame> {
        Some(self.0)
    }
}

fn out(o: &Outputs, i: usize) -> Option<Frame> {
    o[i]
}

#[test]
fn three_party_conference_is_n_minus_one() {
    let mut b = conference(&[0, 1, 2]);
    let o = b.tick(&inputs_with(&[(0, 1000), (1, 2000), (2, 4000)]));
    assert_eq!(out(&o, 0), Some(Frame::filled(6000)));
    assert_eq!(out(&o, 1), Some(Frame::filled(5000)));
    assert_eq!(out(&o, 2), Some(Frame::filled(3000)));
}

#[test]
fn mute_removes_a_speaker_but_not_their_hearing() {
    let mut b = conference(&[0, 1, 2]);
    b.gates_mut().set_muted(s(1), true);
    let o = b.tick(&inputs_with(&[(0, 1000), (1, 2000), (2, 4000)]));
    assert_eq!(out(&o, 0), Some(Frame::filled(4000)));
    assert_eq!(out(&o, 1), Some(Frame::filled(5000))); // still hears 0 and 2
    assert_eq!(out(&o, 2), Some(Frame::filled(1000)));
}

#[test]
fn deafen_silences_the_listener_only() {
    let mut b = conference(&[0, 1, 2]);
    b.gates_mut().set_deafened(s(0), true);
    let o = b.tick(&inputs_with(&[(0, 1000), (1, 2000), (2, 4000)]));
    assert_eq!(out(&o, 0), Some(Frame::silence()));
    assert_eq!(out(&o, 1), Some(Frame::filled(5000)));
}

#[test]
fn taps_see_audio_from_muted_legs() {
    let mut b = conference(&[0, 1]);
    let (tap, handle) = queue_tap(16);
    b.add_tap(Box::new(tap));
    b.gates_mut().set_muted(s(1), true);
    b.tick(&inputs_with(&[(0, 1000), (1, 2000)]));
    let mut seen = Vec::new();
    while let Some(item) = handle.pop() {
        seen.push(item);
    }
    assert!(seen.contains(&(s(1), Frame::filled(2000))));
}

#[test]
fn whisper_reaches_only_the_agent() {
    // 0 = agent, 1 = customer, 3 = supervisor
    let mut b = conference(&[0, 1]);
    b.attach(s(3));
    b.set_pinned(s(3), true);
    b.matrix_mut().set(s(3), s(0), Gain::UNITY).unwrap(); // whisper
    b.matrix_mut().set(s(0), s(3), Gain::UNITY).unwrap(); // listen
    b.matrix_mut().set(s(1), s(3), Gain::UNITY).unwrap();
    let o = b.tick(&inputs_with(&[(0, 1000), (1, 2000), (3, 3000)]));
    assert_eq!(out(&o, 0), Some(Frame::filled(5000))); // customer + supervisor
    assert_eq!(out(&o, 1), Some(Frame::filled(1000))); // agent only
    assert_eq!(out(&o, 3), Some(Frame::filled(3000))); // agent + customer
}

#[test]
fn only_the_loudest_speakers_are_mixed() {
    let values: [i16; 6] = [1000, 2000, 3000, 4000, 5000, 6000];
    let mut b = conference(&[0, 1, 2, 3, 4, 5]);
    let pairs: Vec<(usize, i16)> = values.iter().copied().enumerate().collect();
    let o = b.tick(&inputs_with(&pairs));
    let selected: Vec<usize> = (0..6).rev().take(MIX_TOP_K).collect();
    for sink in 0..6 {
        let expected: i16 = selected
            .iter()
            .filter(|&&i| i != sink)
            .map(|&i| values[i])
            .sum();
        assert_eq!(out(&o, sink), Some(Frame::filled(expected)), "sink {sink}");
    }
}

#[test]
fn hold_source_replaces_a_legs_incoming_audio() {
    // leg 0 is on hold: nobody feeds it, the hold source (slot 5) does.
    let mut b = conference(&[0, 1]);
    b.matrix_mut().set(s(0), s(1), Gain::ZERO).unwrap();
    b.matrix_mut().set(s(1), s(0), Gain::ZERO).unwrap();
    b.attach_source(s(5), Box::new(ConstSource(Frame::filled(500))));
    b.matrix_mut().set(s(5), s(0), Gain::UNITY).unwrap();
    let o = b.tick(&inputs_with(&[(0, 1000), (1, 2000)]));
    assert_eq!(out(&o, 0), Some(Frame::filled(500))); // quiet but pinned
    assert!(out(&o, 1).is_some_and(|f| f.is_silence()));
}

#[test]
fn input_for_a_detached_slot_is_ignored() {
    let mut b = conference(&[0, 1, 2]);
    b.detach(s(1));
    let o = b.tick(&inputs_with(&[(0, 1000), (1, 2000), (2, 4000)]));
    assert_eq!(out(&o, 1), None);
    assert_eq!(out(&o, 0), Some(Frame::filled(4000)));
    assert_eq!(out(&o, 2), Some(Frame::filled(1000)));
}

#[test]
fn attached_legs_without_input_get_silence_not_none() {
    let mut b = conference(&[0, 1]);
    let o = b.tick(&[None; MAX_LEGS_PER_BRIDGE]);
    assert_eq!(out(&o, 0), Some(Frame::silence()));
    assert_eq!(out(&o, 7), None); // never attached
}
```

- [ ] **Step 2: Run, expect FAIL** — `cargo test -p nimossini-core --test bridge`
- [ ] **Step 3: Implement** `Source` (trivial) then `Bridge`. Hints:
  `std::array::from_fn(|_| None)` initializes arrays of non-`Copy` values;
  a `Vec<(Slot, Box<dyn Source>)>` is fine for sources; write `tick` as a few
  small private methods (`gather_inputs`, `levels`, `run_taps`, `apply_mute`,
  `apply_top_k`, `render`) so each is readable and the function stays short.
  Think about `&mut self` borrows: taps need `&mut`, the matrix only needs `&`.
- [ ] **Step 4: Run, expect PASS** — `cargo test`, fmt, clippy
- [ ] **Step 5: Commit** — `git commit -am "feat(core): add Source and Bridge::tick"`

---

### Task 9: `ToneSource` and the listening example

**Concepts:** `f32` trigonometry, phase accumulation, examples, dev-dependencies.

**Files:**
- Modify: `crates/core/src/source.rs`
- Create: `crates/core/examples/three_party.rs`

**Produces:**
```rust
pub struct ToneSource { /* freq_hz, amplitude, phase */ }
impl ToneSource { pub fn new(freq_hz: f32, amplitude: i16) -> Self; }
impl Source for ToneSource { /* always Some(frame), phase-continuous */ }
```

- [ ] **Step 1: Write the failing tests** (in `source.rs`)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn tone_has_the_expected_frequency() {
        let mut t = ToneSource::new(440.0, 8000);
        let mut crossings = 0i32;
        let mut prev = 0i16;
        for _ in 0..50 {
            // 50 frames of 20 ms = 1 second
            let f = t.next_frame().unwrap();
            for &s in f.samples().iter() {
                if prev < 0 && s >= 0 {
                    crossings += 1;
                }
                prev = s;
            }
        }
        assert!((crossings - 440).abs() <= 2, "got {crossings}");
    }

    #[test]
    fn tone_amplitude_is_bounded() {
        let mut t = ToneSource::new(440.0, 8000);
        let f = t.next_frame().unwrap();
        assert!(f.samples().iter().all(|&s| s.abs() <= 8000));
    }

    #[test]
    fn tone_is_continuous_across_frames() {
        let mut t = ToneSource::new(440.0, 8000);
        let a = t.next_frame().unwrap();
        let b = t.next_frame().unwrap();
        let step = (a.samples()[FRAME_SAMPLES - 1] as i32 - b.samples()[0] as i32).abs();
        assert!(step < 1500, "jump of {step} between frames");
    }
}
```

- [ ] **Step 2: Run, expect FAIL** — `cargo test -p nimossini-core source`
- [ ] **Step 3: Implement.** Hint: keep `phase` in radians, advance by
  `2π·f/SAMPLE_RATE_HZ` per sample, wrap with `%= TAU` (`std::f32::consts::TAU`).
- [ ] **Step 4: Write `examples/three_party.rs`:** three `ToneSource`s (e.g. 300,
  440, 600 Hz) feed slots 0-2 of a conference `Bridge` for 250 ticks (5 s);
  write each slot's output to `target/out_{n}.wav` with `hound` (mono, 16-bit,
  16 kHz). Run `cargo run --example three_party` and listen: each file should
  contain the *other two* tones, never its own.
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat(core): add ToneSource and three_party example"`

---

### Task 10: Property tests (Review Focus 2)

**Concepts:** property-based testing with `proptest`, `prop_assert!`.

**Files:** Create `crates/core/tests/properties.rs`

- [ ] **Step 1: Write the tests**

```rust
use nimossini_core::frame::Frame;
use nimossini_core::matrix::Matrix;
use nimossini_core::mixer::{mix_for_sink, Inputs};
use nimossini_core::slot::Slot;
use nimossini_core::MAX_LEGS_PER_BRIDGE;
use proptest::prelude::*;

fn full_mesh() -> Matrix {
    let all: Vec<Slot> = (0..MAX_LEGS_PER_BRIDGE)
        .map(|i| Slot::new(i).unwrap())
        .collect();
    let mut m = Matrix::new();
    m.connect_all(&all);
    m
}

proptest! {
    #[test]
    fn a_sink_never_hears_only_itself(sink in 0..MAX_LEGS_PER_BRIDGE, v in any::<i16>()) {
        let m = full_mesh();
        let mut inputs: Inputs = [None; MAX_LEGS_PER_BRIDGE];
        inputs[sink] = Some(Frame::filled(v));
        let out = mix_for_sink(&m, &inputs, Slot::new(sink).unwrap());
        prop_assert!(out.is_silence());
    }

    #[test]
    fn arbitrary_loud_inputs_never_panic(
        values in prop::collection::vec(any::<i16>(), MAX_LEGS_PER_BRIDGE)
    ) {
        let m = full_mesh();
        let mut inputs: Inputs = [None; MAX_LEGS_PER_BRIDGE];
        for (i, v) in values.iter().enumerate() {
            inputs[i] = Some(Frame::filled(*v));
        }
        for sink in 0..MAX_LEGS_PER_BRIDGE {
            let _ = mix_for_sink(&m, &inputs, Slot::new(sink).unwrap());
        }
    }
}
```

- [ ] **Step 2: Run** — `cargo test -p nimossini-core --test properties`.
  These should pass already if Tasks 2-4 are right. If one fails, `proptest`
  prints a minimal failing input: read it carefully, it is the lesson.
- [ ] **Step 3: Commit** — `git add -A && git commit -m "test(core): add mixer property tests"`

---

### Task 11: First capacity data point

**Concepts:** `std::time::Instant`, release builds, `black_box`.

**Files:** Create `crates/core/examples/tick_cost.rs`

- [ ] **Step 1:** Write an example that builds a 32-leg full conference with a
  `ToneSource`-style input per leg, runs 5,000 ticks, and prints the average and
  maximum duration of `tick` in microseconds. Use `std::hint::black_box` on the
  outputs so the work is not optimized away.
- [ ] **Step 2:** Run `cargo run --release --example tick_cost`. The budget is
  20,000 µs per tick; record what you measure in
  `docs/superpowers/notes/core-tick-cost.md` (hardware, CPU, numbers, and the
  value of `MIX_TOP_K`).
- [ ] **Step 3: Commit** — `git add -A && git commit -m "perf(core): measure tick cost for 32 legs"`

---

## Self-review (against the spec)

- Spec 4.2, 4.3, 4.4 (frames, tick, matrix features): Tasks 1-8. Mute, deafen,
  whisper, hold, and top-K each have a test in Task 8.
- Spec 4.5 (compile-time limit): Tasks 0 and 3 (the experiment in Task 3 step 3).
- Spec 4.6 and 8 (bounded droppable consumers, no blocking): Task 7.
- Spec 12.1 (core unit and property tests): all tasks plus Task 10.
- Deferred to later plans: sample-rate negotiation and codecs (edge), command and
  event channels, VAD beyond the level threshold, the intercept tap module,
  gRPC, and the server. The first capacity data point (Task 11) informs the
  admission-control design.
- These tests have not been compiled by the assistant (Rust is not yet installed
  on this machine). If a test does not compile as written, tell the assistant:
  that is a plan bug, not a mistake of yours.
