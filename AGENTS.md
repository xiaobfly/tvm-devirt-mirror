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

Sample driver: `test/ACE-SSC-DRV64.sys`, kept locally and deliberately not tracked (`.gitignore` ignores `*.sys`). Nothing in the build or the test suite depends on it; the smoke runs below do, so ask the user for a copy if it is missing.

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
tvm-devirt entries test/ACE-SSC-DRV64.sys --all   # 192 proven locations here; --include-stubs restores all 338
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

- Framework: plain `#[cfg(test)] mod tests` in-module (Rust's built-in harness). `cargo test` needs no external fixtures and runs 212 tests in well under a second.
- Coverage is concentrated in the IR and codegen layers: `src/codegen/tests.rs` (child module declared at `codegen/mod.rs:26`, reaches private emitters), `src/ir/expr.rs`, `src/ir/lift.rs`, `src/ir/sched.rs`, `src/ir/regalloc.rs`, `src/vm/explore.rs`, `src/vm/state.rs`, `src/vm/discover.rs`, `src/cli/diag.rs`.
- **Untested**: `src/main.rs`, `inspect.rs`/`recover.rs`/`emit.rs`/`fmt.rs` under `src/cli/`, PE parsing and patch building in `src/binary/`, `src/vm/diag.rs`, `src/ir/hash.rs`. Changes there need a manual smoke run against `test/ACE-SSC-DRV64.sys`.
- Test naming is sentence-style snake_case describing the bug prevented, e.g. `a_spill_store_no_one_reads_is_removed` (`codegen/tests.rs:148`).
- Assertions are text/structural, not golden files: `codegen/tests.rs` assembles with `CodeAssembler` and compares `NasmFormatter` output via helpers `emit_one`, `seg`, `disasm_bytes`; IR tests build synthetic PEs with `executable_pe` / `pe_with` (`ir/lift.rs:3085`, `:3322`).
- Test-only helpers inside production modules are `#[cfg(test)]`-gated — keep them that way.
- Smoke paths verified against the local sample (`test/ACE-SSC-DRV64.sys`, untracked): `entries` lists 192 proven locations by default and 338 with `--include-stubs`; `cfg`/`lower` recover 18 blocks (126 instructions) at `0x140001000`; `devirt` emits 877 bytes of x86 for it. Use these to check a change end-to-end.
- Full-pipeline check: `cargo run -- write-devirt test/ACE-SSC-DRV64.sys out.sys --jobs 8 --timeout 30` reports `321 ready, 17 incomplete` and writes a patched image in ~75 s (`unresolved=92 (block-budget truncated=21, genuine=71)`, families `left-image 64, block-budget 21, argument-dependent 5, work-budget 1, unresolved-branch 1`, checksum `0x0025a81d`). `--max-blocks 1024` gets `324 ready, 14`, and `--max-blocks 1024 --steps 4000000` gets `325 ready, 13` (checksum `0x0028fa36`, `unresolved=70`, all genuine) — at that point every remaining failure is either the runtime-context-pointer family or one of the two structural bodies, i.e. 179 of the 192 census-proven entries. `truncated` is cross-checked against `blocks − budget`, so a budget artefact can never be read as a recovery failure.
- Failure taxonomy of the remaining 17 trampolines (8 distinct VM bodies) — check this before assuming a regression, and prefer the flag over a code change where one exists:
  - **block budget** (3 trampolines, body `0x14008b459`, `truncated=7`): recovers fully at `--max-blocks 1024` (689 blocks, 0 unresolved). Default is 512 on purpose.
  - **per-block step budget** (1, body `0x14007090b`): recovers at `--steps 4000000` (11 blocks, 0 unresolved); the single block needs >500k steps.
  - **runtime-keyed dispatch** (11, bodies `0x140076505`, `0x14006a0f1`, `0x140015ac0`, `0x14008a85e`): every `left image` site is `0x140162f40`, whose handler is `lea rcx,[1400D8AD5h]; movsxd rdx,edx; jmp rax` with `rax = 0x1400D8AD5 + sext32(((X|0x15F0D388) + K) ^ 0x3275DE05)` — inline immediates only, no table read at the site, so the masks are independent of any data value. The unknown is `K`/`X`, which trace back to `load64[0x14001a598]`: a `.data` qword that is **zero in the file and has no writer anywhere in the image** (five RIP-relative references, all reads: `0x14008a86d`, `0x14008a941`, `0x14008ab68`, `0x14008ac32`, `0x14009de60`, each dereferenced as a context base at `+4`/`+0x188`/`+0x11B8`/`+0x21D0` with null guards). It is a runtime context pointer installed by something outside the protected image, not key material. Same root cause for the `unresolved branch (0 candidates)` through `load64[(load64[0x14001a598] + 0x11b8)]`. Not statically computable; do not "fix" the fold. The mechanical reason the load stays symbolic is policy (`ir/lift.rs:1085-1100` refuses writable-section reads unless BSS, and `.data` here is file-backed).
  - **structural judgments** (2, bodies `0x14007cfd6`, `0x1400a9ee3`) — investigated to the bottom; neither is a missing-mechanism patch, and both refusals stand:
    - `unidentified 7` (`0x1400a9ee3`): the block ids are fixed at forks long before any context exists, and `vip_slot` is `None` because the 20 000-step VIP probe dies at the first symbolic branch (`sampled=0`, step 20). The prologue spills do not establish a context either — 1 and 2 register stores against `MIN_SPILL_REGS = 6`. Frame-wide sampling across the whole recovery finds in-VM values at `-0x2a0` (73 distinct), but forcing that slot leaves `unidentified 7` for every offset tried.
    - `cannot_reach_exit 3` (`0x14007cfd6`): forcing a synthetic context yields 32 phi nodes (params do form) yet the same 3-block cycle on all 25 candidate bases, so missing params are not the blocker; frame-wide sampling finds **zero** in-VM values anywhere in its frame over the whole 196k-step recovery, i.e. there is no bytecode position to identify those blocks with.
    - Context discovery is not a prerequisite for correct output: `0x140001000` also runs with `vip_slot=None` and retargets to the same 877 bytes. Context matters for block identity and for parameters, not for correctness of the emitted code as such.
    - Why deferred re-identification is not worth doing on the evidence: the ids are forged at the fork before any state exists, `vip_slot` is `None` for the whole function, and forcing a candidate frame slot (`-0x2a0` carried 73 distinct in-VM values) changed the graph but still left `unidentified 7` for every offset tried. A real fix has to discover the slot post-hoc, attach the identity after the block body runs, and then re-key `visited`, the pending queue, `edges`, `back_edges` and `preds` — core-machinery surgery for one trampoline out of 338, with no measured evidence that it clears the flag, and a naive version carries the successor's VIP and corrupts back-edge matching. Recorded as a dead end, not a backlog item.
  - The runtime-context pointer was also ruled out by recovery itself, not only by reading bytes: `cfg --events` over all 338 trampolines shows zero store events to `0x14001a598` (only loads and stores *of* the loaded value elsewhere), so nothing inside the protected image installs it.
- A worker panic during any sweep means an SSA-scoping regression — that is what `src/ir/sched.rs:291-296` exists to catch; never delete the assertion.
