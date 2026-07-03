# YOLO TFMA INT8 — Recipe

Ports the 1x1 conv in the scored benchmark kernel
(`ported_models/yolo/src/yolo_vpu_argbuf.c`) from FP32 VPU to INT8 TFMA,
behind `-DYOLO_TFMA_INT8`. Note: the scored kernel is a synthetic benchmark
(`CH=16`, weights generated on-chip by `init_model()`) — not real YOLOv10n,
no ONNX involved.

## Approach

1. **Weights** — quantized once on-chip, per-output-channel scale, straight
   from `init_model()`'s integer weights (no host tooling needed).
2. **Activations** — quantized dynamically per-hart row-stripe before each
   dispatch (per-tensor scale = max_abs/127).
3. **Dispatch** — `tensor_load` (weights → TenB, activations → SCP) →
   `tensor_fma(opcode=3)` → `tensor_store` → dequantize INT32 → FP32 + relu6.
4. **Thread gating (the key fix)** — the tensor unit (TFMA/tload/tstore) is a
   per-*minion* resource shared by both hardware threads, confirmed by
   reading the sys-emu source (`Core` struct + `Hart::async_execute()` only
   services pending tensor ops on thread 0). With `ACTIVE_HARTS=16` half the
   harts are thread-1, so calling TFMA unconditionally would race or hang.
   Fixed: TFMA only runs on thread-0 harts; thread-1 keeps the proven FP32
   VPU path for its own row range.

## Key references

- `et-platform/et-common-libs/include/erbium/isa/tensors.h` — CSR encodings
  for `tensor_load`/`tensor_fma`/`tensor_store`/`tensor_wait`.
- `et-platform/sw-sysemu/processor.h`, `insns/tensors.cpp` — settled the
  thread-safety question above from emulator source, not guesswork.
- `ported_models/yolo/docs/optimizations.md` — the M18/M19/M21/M22 "silicon
  flake" entries are the same bug class (shared resource, no T0 gate).

## Commands

```bash
GCC=$ET_INSTALL/bin/riscv64-unknown-elf-gcc ./scripts/build_yolo_10.sh
scripts/run_sysemu_model_ports.sh --launcher "$LOCAL_ARGBUF" --suite smoke   # local, no board needed
./scripts/run_yolo_10.sh
grep -H "Kernel wait" run_y10_*.log | sort -t: -k3 -n                        # compare vs y10_00_base 0.134534s
```

## Verification

- Compiles clean C (`-nostdlib`, no C++ runtime on this target).
- `get_tensor_error()` captured once into `summary->reserved[0]` — should
  read 0.
- `output_sum` will **not** match the FP32 baseline bit-for-bit (INT8 is
  lossy by design) — expected, not a failure. Leaderboard scores kernel wait
  only.

## Dead end

`tl_tfma_2s.c` (an etsoc reference kernel) calls `drain_scb()` after
`tensor_store`. That function doesn't exist for `erbium`/`erbium-soc1sim`.
Checked the emulator: `tensor_store_execute()` is a normal memory-mapped
store, so the existing `tensor_wait` + `evict` + `WAIT_CACHEOPS` pattern
already used throughout this file is sufficient — no drain needed.
