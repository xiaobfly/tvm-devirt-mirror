# Repository Guidelines

## Project Overview

`tvm-devirt` — a static devirtualizer for Tencent VM (TVM) protected x64 PE binaries. It finds virtualized entry trampolines, symbolically emulates the VM dispatcher, recovers per-function CFGs/SSA, schedules them to three-address code, and emits native x86 that is patched back into a copy of the input. Output is **analysis-grade**, not production-quality (`README.md:94-95`).

Single binary crate, no library: `src/main.rs` → `tvm-devirt`. MIT (`LICENSE:1`), edition 2024 ⇒ Rust ≥ 1.85.

## Architecture & Data Flow

```
entries  →  (trace | cfg)  →  ssa  →  lower  →  devirt | write-devirt
discover    emulator + symbolic forking   phi/param   schedule   codegen + PE patch
```

1. `vm/discover.rs` scans executable sections for `E9 rel32` jumps landing in the VM section (`.tvm0`) and rejects false positives at `.pdata` boundaries.
2. `vm/explore.rs` runs a two-pass recovery (shape, then join-only parameter cutting) with a per-path instruction budget; it forks the evaluator when a branch depends on an unresolved guest value (`split`, `split_multiway`).
3. `ir/lift.rs` emulates the dispatcher, lifts handler effects into hash-consed expressions (`ir/expr.rs`), and locates the guest register image by scoring candidate context bases rather than assuming one layout.
4. `ir/sched.rs` linearizes blocks; `ir/regalloc.rs` performs linear-scan allocation with ABI clobbers at calls.
5. `codegen/` emits raw x64 via iced-x86 `CodeAssembler`; `binary/writeback.rs` appends the new section, fixes up exception data and checksum, and returns bytes (it never touches the filesystem).

Key identity rule: a VM block is `(handler, vip)`; the VIP is an FNV-1a hash of the VIP slot plus the 0xa8-byte context window (`ir/lift.rs:508-540`).

SSA scoping rule: an `Op::Param(owner, reg)` may only be used by the block it belongs to — it means "value of `reg` on entry to `owner`". Recovery cuts parameters only at the joins pass 1 found, so a pass-2 join or an unlocatable guest context leaves descendants reaching for an ancestor's parameter; `Cfg::localize_params` (`ir/ir.rs`, called from `Cfg::recover` in `vm/explore.rs:389`) rewrites those into parameters of the using block, which `Cfg::phis` then feeds from each predecessor's `exit_regs`. `src/ir/sched.rs:291-296` debug-asserts the invariant.

Recovery diagnostics: `vm/diag.rs` is an opt-in thread-local collector (pin/flag/switch probes, stops, compactions, divergence), and `RecoveryDeadline` subtracts the time spent inside it so measuring never changes what is measured. `UnresolvedStats` (`vm/explore.rs`) buckets every `Terminator::Unresolved` reason into families and separates budget truncation from genuine failure — `truncated` (blocks carrying `block budget reached`) is checked against the arithmetic prediction `blocks − budget`, and a mismatch is reported rather than hidden. `cli/diag.rs` exposes all of it as the `unresolved` subcommand, with `--json` for a machine-readable report.

## Key Directories

| Path | Purpose |
|---|---|
| `src/binary/` | Hand-rolled PE32+ reader (`pe.rs`), format constants (`pe_format.rs`), iced-x86 decode wrappers (`disasm.rs`), pure patch builder (`writeback.rs`) |
| `src/vm/` | Entry discovery + stub census (`discover.rs`), guest machine state (`state.rs`), CFG exploration + `UnresolvedStats` (`explore.rs`), opt-in diagnostics sink (`diag.rs`) |
| `src/ir/` | Core IR + terminators (`ir.rs`), hash-consed expression arena (`expr.rs`), FNV (`hash.rs`), emulator/lifter (`lift.rs`, ~3900 lines), scheduling (`sched.rs`), allocation (`regalloc.rs`) |
| `src/codegen/` | Instruction lowering (`emit.rs`), branches/terminators (`control.rs`), frame+validation (`function.rs`), layout (`layout.rs`), operand mapping (`operands.rs`), Win64 unwind (`unwind.rs`), unit tests (`tests.rs`) |
| `src/cli/` | `inspect.rs`, `recover.rs` (dump-only), `emit.rs` (only place that writes files), `fmt.rs` (shared printers), `diag.rs` (the `unresolved` subcommand) |
| `src/vm/` | Entry discovery + stub census (`discover.rs`), guest machine state (`state.rs`), CFG exploration + `UnresolvedStats` (`explore.rs`), opt-in diagnostics sink (`diag.rs`) |
| `docs/` | Mirror design/plan/validation notes for the arena-compaction work and the diagnostics |

No CI, no `tests/`/`benches/`, no scripts, no formatter/linter config anywhere in the repo.

Sample driver: not in version control. The local copy currently sits at `target/release/test/ACE-SSC-DRV64.sys`, inside the gitignored `target/`, so `cargo clean` takes it with it — ask the user for another copy if it is gone.

Five files hold most of the ~26k lines: `ir/lift.rs` (3933), `ir/expr.rs` (3774), `ir/sched.rs` (3708), `codegen/tests.rs` (2318), `vm/explore.rs` (2243). Read them in ranges; grep for the function name first.

## Development Commands

All verified against the built binary:

```bash
cargo build              # dev profile is opt-level = 1, builds fast enough to iterate
cargo build --release    # debug = true, keeps symbols for the emitted x64
cargo test               # unit tests only, no fixtures required
cargo run -- <cmd> ...   # or ./target/debug/tvm-devirt.exe <cmd> ...
```

```bash
tvm-devirt entries target/release/test/ACE-SSC-DRV64.sys --all   # 192 proven locations here; --include-stubs restores all 338
tvm-devirt trace  <input> <va> [--events] [--dispatches] [--tail N] [--break-at VA]
tvm-devirt cfg    <input> <va> [--events] [--state] [--steps N] [--max-blocks N] [--timeout S]
tvm-devirt ssa    <input> <va> [--joins-only]
tvm-devirt lower  <input> <va>
tvm-devirt devirt <input> <va> [--output file.bin] [--dis] [--max-blocks N]
tvm-devirt write-devirt <input> <output> [--jobs N] [--steps N] [--timeout S] [--max-blocks N]
tvm-devirt unresolved <input> <va> [--steps N] [--max-blocks N] [--timeout S] [--json FILE]
```

Also `info`, `dis --count N`, `vmctx --window N`. `--vm-section` (default `.tvm0`) exists on `info`, `entries`, `cfg`, `ssa`, `lower`, `write-devirt` only; `--max-blocks` defaults to 512 on `cfg`, `devirt`, `write-devirt` and `unresolved`, 256 on `ssa` and `lower`; `--stack-base` defaults to `0x7fff_fffe_fda0`; VAs accept `0x` prefixes.

### Debug env knobs (undocumented in README)

`TVM_MAP`, `TVM_IMPORT=<va>`, `TVM_ALL_IMPORTS`, `TVM_FULL_ADDR`, `TVM_SHOW_ALLOC`, `TVM_STRICT_FRAME`, `TVM_NODE_LIMIT`, `TVM_ALLOC_ERRS`.

## Code Conventions & Common Patterns

- **Errors**: `anyhow::Result` everywhere, with `bail!` / `ensure!` / `.context()`. No custom error enum (only `AllocError`, dead-code-gated).
- **Panics** are reserved for internal invariants (`expect("…")`, `unreachable!()`, `debug_assert_eq!`) — never for user input.
- **Safety**: zero `unsafe`, zero `extern` in `src/`. Little-endian byte handling throughout; FNV-1a via `ir/hash.rs`.
- **IR construction**: never build `Node`s by hand — go through the folding constructors on `Arena` (`ir/expr.rs:602-703`) so hash-consing and `known_zero`/`known_one` folding stay sound. `Ref` is a 1-based `NonZeroU32`; cells are 8-byte aligned/strided.
- **Purity**: everything under `src/codegen/` and `src/binary/` is IO-free — no `println!`, no `std::fs`. Result printing lives in `src/cli/`.
- **Output streams**: results to stdout, progress/diagnostics to stderr (`binary/writeback.rs:126-135`).
- **Naming**: verb-first `snake_case` (`recover_pass_with`, `drop_unreachable`, `narrow_partially_dead_stores`); `with_*` alternate constructors; `is_*`/`may_*` predicates; full ordering derives on IDs (`Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord`), `Clone` only on large types (`Arena` has a hand-written `Clone`).
- **Docs**: `//!` on modules states rationale and the invariant the code protects; `///` on every public item. Keep that style — it is how the recovery invariants are recorded.
- **Do not reformat**: there is no `rustfmt.toml`, so `cargo fmt` would churn files. Match surrounding code.

## Important Files

| File | Why it matters |
|---|---|
| `src/main.rs` | clap `Cli`/`Cmd` definition (11 subcommands), dispatch into `cli::*`, `parse_u64` VA parser |
| `src/vm/mod.rs` | `DEFAULT_STACK_BASE`, `DEFAULT_VM_SECTION` |
| `src/vm/diag.rs` | Thread-local diagnostics sink: `begin`/`set_pass`/`record_*`/`take`; `excluded_time` feeds the recovery deadline |
| `src/cli/diag.rs` | `cmd_unresolved` — the JSON report behind the `unresolved` subcommand |
| `src/vm/explore.rs` | `Explorer` + `Cfg::recover` (calls `localize_params`), `UnresolvedStats`, `validate` / `CfgIssues` |
| `src/ir/ir.rs` | `Cfg`, `Block`, `Terminator`, `phis`/`simplify_phis`/`localize_params` — the document every pass consumes |
| `src/ir/lift.rs` | Emulator, `Event::{Store, Load, Boxed, Call}`, guest-context location |
| `src/ir/expr.rs` | `Arena`, `Op`, folding constructors, `structural_hash` (arena-stable, ignores `Load` generation) |
| `src/codegen/function.rs` | `plan_frame` + `unwind_info` contract, `TVM_STRICT_FRAME` gate |
| `src/binary/writeback.rs` | `write_devirt(...) -> Result<(Vec<u8>, DevirtReport)>` — patch builder, first-2-GiB trampoline limit, signature dropping |
| `src/cli/emit.rs` | The only two `std::fs::write` calls in the crate |

## Runtime/Tooling Preferences

- Cargo only; no toolchain file, no `.cargo/config.toml`, no workspace. Binary name is literally `tvm-devirt`.
- Dependencies are deliberately minimal: `anyhow`, `clap` (derive), `iced-x86` (`code_asm` feature) for both decoding and assembling, `rayon` for concurrent per-function recovery. There is **no** PE crate — parsing is hand-rolled in `src/binary/pe.rs`; do not introduce `goblin`/`object` without reason.
- `[profile.dev] opt-level = 1` is intentional (some tests are timing/deadline-sensitive); `[profile.release] debug = true` keeps data available for RE work.
- Inputs must be PE32+ AMD64 with a `.tvm0`-style VM section; non-PE or 32-bit inputs are rejected at `pe.rs:51-62`.
- `write-devirt` overwrites `<output>` with no backup and has **no guard against `output == input`** — verify paths before running it on a real sample.

## Testing & QA

- Framework: plain `#[cfg(test)] mod tests` in-module (Rust's built-in harness). `cargo test` needs no external fixtures and runs 208 tests in well under a second.
- Coverage is concentrated in the IR and codegen layers: `src/codegen/tests.rs` (child module declared at `codegen/mod.rs:26`, reaches private emitters), `src/ir/expr.rs`, `src/ir/lift.rs`, `src/ir/sched.rs`, `src/ir/regalloc.rs`, `src/vm/explore.rs`, `src/vm/state.rs`, `src/vm/discover.rs`, `src/cli/diag.rs`.
- **Untested**: `src/main.rs`, `inspect.rs`/`recover.rs`/`emit.rs`/`fmt.rs` under `src/cli/`, PE parsing and patch building in `src/binary/`, `src/vm/diag.rs`, `src/ir/hash.rs`. Changes there need a manual smoke run against `target/release/test/ACE-SSC-DRV64.sys`.
- Test naming is sentence-style snake_case describing the bug prevented, e.g. `a_spill_store_no_one_reads_is_removed` (`codegen/tests.rs:148`).
- Assertions are text/structural, not golden files: `codegen/tests.rs` assembles with `CodeAssembler` and compares `NasmFormatter` output via helpers `emit_one`, `seg`, `disasm_bytes`; IR tests build synthetic PEs with `executable_pe` / `pe_with` (`ir/lift.rs:3085`, `:3322`).
- Test-only helpers inside production modules are `#[cfg(test)]`-gated — keep them that way.
- Smoke paths verified against the local sample (`target/release/test/ACE-SSC-DRV64.sys`, untracked): `entries` lists 192 proven locations by default and 338 with `--include-stubs`; `cfg`/`lower` recover 18 blocks (126 instructions) at `0x140001000`; `devirt` emits 877 bytes of x86 for it. Use these to check a change end-to-end.
- Full-pipeline check: `cargo run -- write-devirt target/release/test/ACE-SSC-DRV64.sys out.sys --jobs 8 --timeout 30` reports `312 ready, 26 incomplete`, retargets 312 of 338 functions in ~85 s and writes a patched image; `--max-blocks 1024` gets `315 ready, 23` because the trampolines `0x14000a9e0`, `0x14000aa40` and `0x14000b681` share VM entry `0x14008b459`, which needs 689 blocks against the default budget of 512 (`prepare_one` in `src/binary/writeback.rs`). The sweep ends with the `UnresolvedStats` accounting (`unresolved=102 (block-budget truncated=21, genuine=81)`) plus the reason families (`left-image`, `block-budget`, `argument-dependent`, `unsupported`, `work-budget`, `unresolved-branch`) — `truncated` is cross-checked against `blocks − budget`, so a budget artefact can never be read as a recovery failure. A worker panic there means an SSA-scoping regression — that is what `src/ir/sched.rs:291-296` exists to catch; never delete the assertion.
