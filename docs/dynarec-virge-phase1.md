# Phase 1: 386 dynarec / S3 ViRGE

This change keeps existing emulation semantics intact and limits Phase 1 to optional instrumentation that is disabled by default.

## Reviewed architecture

- `src/cpu/386_dynarec.c`
  - `exec_interpreter()` runs whole blocks through `x86_opcodes[...]` and exits on page changes, `abrt`, `trap`, SMI/NMI, or explicit block-end conditions.
  - `exec_recompiler()` does a fast `codeblock_hash` lookup, validates `pc/_cs/phys/status`, falls back to `codeblock_tree_find()` on misses, checks `dirty_mask/page_mask`, and then either recompiles, executes, or marks blocks.
  - During recompilation, every instruction still passes through the classic handler path `x86_opcodes[...]` after `codegen_generate_call(...)`.
- `includes/private/codegen/codegen.h`
  - `codeblock_t` carries `pc`, `phys`, `status`, `flags`, per-page masks (`page_mask`, `page_mask2`), dirty-mask pointers, and both per-page list links and tree links.
- `src/codegen/codegen_block.c`
  - `codegen_check_flush()` invalidates blocks when `dirty_mask & page_mask` intersects.
  - `codegen_block_init()`, `codegen_block_start_recompile()`, `codegen_block_end()`, and `codegen_block_end_recompile()` are the safe lifecycle points for instrumentation.
- `src/codegen/codegen.c` and `src/codegen/codegen_x86-64.c`
  - `codegen_generate_call()` decides between a native dynarec translator (`recomp_op_table[...]`) and a fallback C handler call.
- `src/cpu/cpu.c`
  - `x86_setopcodes()` wires together `x86_opcodes` and `x86_dynarec_opcodes`.
- `src/video/vid_s3_virge.c`
  - `s3_virge_bitblt()` contains BitBLT, rectfill, line, and poly dispatch.
  - `MIX()` applies the ROP bit-by-bit.
  - `tri()` and `s3_virge_triangle()` drive 3D rasterization, VRAM accesses, and texture sampling.

## Bottlenecks and compatibility risks

- The current dynarec cannot be assumed to be “mostly native”: the real native-vs-C split only happens inside `codegen_generate_call()` and depends on the exact opcode/table combination.
- Block validation depends on:
  - current `pc/_cs/phys/status`;
  - page dirtiness and per-page masks;
  - recompilation on static FPU `TOP` mismatches.
  Because of that, this phase does **not** force a last-block cache or similar shortcut without first measuring those cases.
- Code invalidation is tightly coupled to `mem_flush_write_page()`, `dirty_mask`, `code_present_mask`, and the `codeblock_t` list/tree structures; an unsafe fast path here would risk breaking self-modifying code.
- In ViRGE, BitBLT and triangle paths mix clipping, copy direction, pixel-format handling, and direct VRAM traffic. Optimizing `MIX()` or bulk-copy behavior without specialization would be risky.

## Added instrumentation

Build with:

```sh
cmake -S . -B build -DPCEM_PERF_STATS=ON
cmake --build build -j
```

When `PCEM_PERF_STATS=ON`, shutdown logging now dumps:

- 386 dynarec:
  - interpreted blocks;
  - recompiler entries;
  - successful fast validations;
  - hash-cache misses;
  - slow validations and slow hits;
  - dirty-page checks;
  - FPU `TOP` recompiles;
  - compiled, executed, and mark-only blocks;
  - block invalidations;
  - exception handling events;
  - native translation decisions vs fallback handler calls.
- S3 ViRGE:
  - BitBLT operations and pixels;
  - rectfill operations and pixels;
  - line/poly operations;
  - triangles, rasterized pixels, and texture samples;
  - VRAM reads/writes observed on these paths;
  - scalar operation count;
  - accumulated CPU-time ticks;
  - per-ROP usage histogram;
  - `avx2_ops`, used by the current guarded ViRGE AVX2 rectfill path when that narrow fast path is selected.

## Baseline benchmark procedure

There is no automated benchmark or test harness in this tree, and no portable ROM/workload fixtures are bundled, so a complete benchmark run could not be executed in the agent environment.

Reproducible commands prepared for contributors (not executed here):

```sh
cmake -S . -B build-rel -DCMAKE_BUILD_TYPE=RelWithDebInfo -DPCEM_PERF_STATS=ON
cmake --build build-rel -j
./build-rel/src/pcem
```

Manual baseline procedure (not executed here):

1. Open the same 386 machine with dynarec enabled, and the same S3 ViRGE configuration.
2. Run the same DOS/Windows 9x/game workload for a fixed window.
3. Close PCem and collect the `pclog`.
4. Compare:
   - `native_translated_instructions` vs `handler_calls`;
   - `cache_misses`, `slow_validations`, `block_invalidations`;
   - `bitblt_pixels`, `triangles`, `rasterized_pixels`, `rop[...]`.

## Current limitations

- This phase does **not** add a block-cache optimization or broad semantic changes. It now includes a narrow, runtime-guarded AVX2 fast path for a ViRGE rectfill/PATCOPY 8/16bpp case, with the existing scalar implementation retained as the fallback for all other cases.
- The proportion of C handlers is only measurable when `PCEM_PERF_STATS=ON` and real workloads actually trigger block recompilation.
- No performance improvement is claimed in this document.
