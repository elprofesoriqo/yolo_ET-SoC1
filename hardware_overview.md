# ET-SoC1 Hardware Overview & Technology Reference

---

## What is this thing?

You are targeting an **ET-SoC1** board — a real custom silicon chip made by AIFoundry / Ainekko, designed to run AI workloads. The processor inside is called **Erbium**. It is based on the **RISC-V** instruction set (specifically `rv64imfc/lp64f`), which means it is a 64-bit open-standard CPU architecture — not x86, not ARM. Think of RISC-V as Lego bricks for CPU designers: a minimal, clean instruction set that anyone can extend.

The chip has special hardware bolted on top of the standard RISC-V core to accelerate AI inference:
- A **VPU** (Vector Processing Unit) for floating-point vector math.
- A **TFMA** (Tensor Function Matmul Accelerator) for integer matrix multiplication.
- A **Scratchpad** (L1 SCP) — a small, private, ultra-fast memory region that is directly managed by your kernel.

---

## Architecture Diagram

```
ET-SoC1 Chip
│
├── 8 Minions (compute clusters)
│   └── Each Minion has:
│       ├── T0 hart (VPU-capable)  ← Thread 0, has RISC-V + VPU
│       └── T1 hart (scalar only)  ← Thread 1, has RISC-V only
│
├── VPU (Vector Processing Unit)
│   └── 8-lane FP32 SIMD  (8 × 32-bit floats per instruction)
│
├── TFMA (Tensor Function Matmul Accelerator)
│   └── Systolic INT8/FP16/FP32 matrix multiplier
│
├── L1 Scratchpad (per core, 48 cache-line slots × 64 bytes = 3072 bytes)
│
└── DRAM (shared, 80 MB buffer for this model)
```

**Total harts:** 16 (8 T0 + 8 T1, one per minion each)

---

## Key Concepts You Must Understand

### Hart
A **hardware thread** — essentially a CPU core that runs your code. You have 16 of them. T0 harts have access to VPU and TFMA. T1 harts can only do scalar math.

### VPU — Vector Processing Unit
The VPU operates on 8 floating-point numbers at once using 256-bit wide registers.

**Key instructions used in the YOLO kernel:**
| Instruction | What it does |
|---|---|
| `fbcx.ps f0, zero` | Fill all 8 lanes of `f0` with 0.0f (initialize accumulator) |
| `flq2 f1, 0(ptr)` | Load 8 floats (256 bits = 32 bytes) from memory into `f1` |
| `fmadd.ps f0, f1, f2, f0` | `f0 = f1 * f2 + f0` for all 8 lanes simultaneously |
| `fsq2 f0, 0(tmp)` | Store 8 floats from `f0` back to memory |

These appear in `yolo_vpu_argbuf.c` in functions like `vpu_dot16_f32()` (which processes 16 input channels = 2 VPU loads of 8).

### TFMA — Tensor Function Matmul Accelerator
This is the big gun. It is a **systolic array** — a grid of multiplier-adder units that are hardwired to stream data through in a pipeline. It is purpose-built for computing `C = A × B` where:
- **A** = activations (pixel values), loaded row by row
- **B** = weights (filter values), loaded in a special layout
- **C** = output accumulator, stored internally in TenC registers, then flushed to FREGs or DRAM

The key difference from VPU: TFMA computes **an entire tile of the output matrix** in one dispatch instead of computing one output element at a time. For a 1x1 convolution with 16 input channels and 16 output channels, VPU processes one output pixel at a time, while TFMA processes a whole tile (e.g., 14 rows × 16 columns) at once.

**TFMA modes (from `tensors.cpp`):**
| opcode | Type | Accumulates as |
|---|---|---|
| 0 | FP32 | float32 |
| 1 | FP16 | float32 |
| **3** | **INT8** | **int32** |

**Why opcode 3?** When you call `tensor_fma(..., opcode=3)`, the hardware reads each "A" element as 4 consecutive INT8 bytes and each "B" element as 4 consecutive INT8 bytes, computes the dot products, and adds them into a 32-bit integer accumulator. This is 4x more compute density per memory byte than FP32.

### L1 Scratchpad (SCP)
The scratchpad is a special region of the L1 cache that is **not evicted automatically** — you control what goes in and what goes out. It is used by the TFMA as its local working memory.

**Dimensions from `cache.h`:**
- `L1_SCP_NUM_SETS = 12`
- `L1_SCP_NUM_WAYS = 4`
- `L1_SCP_ENTRIES = 48` (12 × 4 = 48 cache-line slots)
- `L1D_LINE_SIZE = 64 bytes` per slot

**Total SCP size = 48 × 64 = 3072 bytes = 3 KB per hart**

The TFMA uses scratchpad slots as row indices. When you call `tensor_load(dst_start=N, ...)`, row `N` of your loaded data goes into SCP slot `N`. When `tensor_fma(astart=N, ...)` is called, it reads rows from SCP starting at slot `N`.

### Cache Ops
Because the chip has non-coherent L1 caches between harts, you must manually flush data so that other harts can see it.

| Macro | What it does |
|---|---|
| `FENCE` | Memory barrier — all prior writes are ordered before this point |
| `evict(ptr, bytes)` | Flush a range of your L1 cache lines to shared memory |
| `WAIT_CACHEOPS` | Stall until all pending evicts are complete |

---

## Key Files in This Repository

### `ported_models/yolo/src/yolo_vpu_argbuf.c`
**The kernel you edit.** This C file is compiled to a RISC-V binary that runs entirely on the chip. It does:
1. Reads weights and activations from DRAM via the memory offsets defined at the top.
2. Computes the YOLO conv layers using VPU intrinsics.
3. Writes a summary struct back to DRAM that the host reads.

Currently all convolutions are FP32 VPU. This is what you are changing.

### `ported_models/yolo/docs/optimizations.md`
**The engineering journal.** Every milestone (M0–M30) of the prior author's work is logged here — what they tried, what failed, what the root cause was, and the final wall time. Read this to understand lessons already learned (e.g. the FREG clobber bug with `tensor_fma` on the DnCNN model).

### `et-platform/et-common-libs/include/erbium/isa/tensors.h`
**The TFMA API header.** Contains the C inline functions you call to issue hardware tensor instructions. Key functions:
- `tensor_load(use_tmask, use_coop, dst_start, transformation, use_tenb, addr, offset, num_lines, stride, id)` — loads rows from DRAM into SCP slots.
- `tensor_fma(use_tmask, b_num_col, a_num_rows, a_num_cols, offset, tenc_loc, tenb_unsigned, tena_unsigned, tenb_loc, scp_loc_b, scp_loc_a, opcode, first_pass)` — fires the matmul.
- `tensor_store(reg_stride, start_reg, cols, Arows, addr, coop_store, stride)` — writes accumulated result from FREGs/TenC back to DRAM.
- `tensor_wait(id)` — waits for async load or store to complete.

### `et-platform/sw-sysemu/insns/tensors.cpp`
**The software emulator of the TFMA hardware.** This is the gold standard for understanding exactly how the hardware works. When you call `tensor_fma` with `opcode=3` (INT8), the emulator runs `tensor_ima8a32_execute()`. The key excerpt shows:

```cpp
// INT8: reads 4 bytes at a time from SCP ("A" matrix rows)
int32_t a1 = ASRC(0);  // byte 0 of the IC-quartet
int32_t a2 = ASRC(1);  // byte 1
int32_t a3 = ASRC(2);  // byte 2
int32_t a4 = ASRC(3);  // byte 3
// "B" matrix columns — also 4 bytes interleaved
int32_t b1 = BSRC(0);
...
int32_t c = (a1*b1) + (a2*b2) + (a3*b3) + (a4*b4);
```

This tells you that your **A-side data (activations)** must be packed so that 4 consecutive input channels for the same spatial position are laid out as 4 consecutive bytes in memory, and your **B-side data (weights)** must be interleaved in the same 4-quartet pattern.

### `et-platform/sw-sysemu/cache.h`
**Hardware constants.** Defines `L1D_LINE_SIZE = 64`, `L1_SCP_ENTRIES = 48`. These numbers govern your tile sizes.

### `ported_models/dncnn/src/tensor_smoke_argbuf.c`
**A working TFMA example.** Proves that `tensor_load` + `tensor_wait` + `tensor_fma` + `tensor_store` work end-to-end on this chip. Study this as a minimal template before scaling up to YOLO's complexity.

---

## Memory Map (YOLO Kernel)

This is the layout of the shared DRAM buffer the kernel uses. Offsets are from `buffer_base`:

| Offset | Symbol | Contents |
|---|---|---|
| `0x0000` | `SLOTS_OFFSET` | Per-hart result slots (64 bytes each) |
| `0x1000` | `SUMMARY_OFFSET` | Global result summary written by hart 0 |
| `0x1800` | `BARRIER_OFFSET` | Software barrier state (epoch counter) |
| `0x4000` | `INPUT_OFFSET` | FP32 input activations [80×80×16] |
| `0x70000` | `WEIGHTS_OFFSET` | FP32 weights (3x3 and 1x1, all blocks) |
| `0x80000` | `ACT0_OFFSET` | FP32 activation ping buffer |
| `0xF0000` | `ACT1_OFFSET` | FP32 activation pong buffer |
| `0x160000` | `OUTPUT_OFFSET` | Final uint8 output |
| `0x1D0000` | `WEIGHTS_INT8_OFFSET` | **[NEW]** Packed INT8 weights for TFMA |
| `0x1F0000` | `SCALE_OFFSET` | **[NEW]** Per-OC FP32 dequant scales |
| `0x200000` | `ACT_INT8_OFFSET` | **[NEW]** Quantized INT8 activations buffer |
