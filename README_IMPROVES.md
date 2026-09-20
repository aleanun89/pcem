# README_IMPROVES

This document summarizes the safest next steps after the Phase 1 dynarec/S3 ViRGE instrumentation work.

## Goal

Keep improvements incremental, measurable, and reversible:

1. preserve emulation correctness first;
2. measure before changing hot paths;
3. keep the old/scalar path available as fallback;
4. avoid host-specific assumptions in common code.

## Recommended next steps

### 1. Gather real baseline data

Before changing execution paths, build with performance counters enabled and capture logs from repeatable workloads:

- a 386 machine with dynarec enabled;
- a guest and workload that exercises S3 ViRGE BitBLT and 3D paths;
- the same ROMs, machine config, guest configuration, and run duration each time.

The first measurements to compare are:

- `native_translated_instructions`
- `handler_calls`
- `cache_misses`
- `slow_validations`
- `block_invalidations`
- `bitblt_ops`
- `bitblt_pixels`
- `triangles`
- `rasterized_pixels`
- `rop[...]`

### 2. Dynarec Phase 2 candidates

Only after baseline data exists, the safest follow-up work is:

- add a conservative last-block or sequential-block cache;
- keep full validation as mandatory fallback;
- validate `pc`, `cs`, `phys`, `status`, dirty pages, and FPU `TOP` exactly as today;
- measure hit rate before attempting deeper translation work.

Do not skip:

- page invalidation logic;
- self-modifying code detection;
- exception/abort paths;
- interrupt-sensitive behavior.

### 3. Dynarec Phase 3 candidates

If measurements show enough handler-heavy traffic and equivalence can be proven, then consider native translation only for low-risk instruction classes first:

- simple register-register ALU ops;
- simple register-immediate ALU ops;
- simple register moves;
- direct control-flow cases that already fit the existing block model.

Keep the current handler path for:

- memory operations with fault risk;
- segment-sensitive behavior;
- I/O;
- FPU/MMX/SSE;
- uncommon or difficult control-flow cases.

### 4. S3 ViRGE follow-up

Use the new counters to determine whether the main payoff is in:

- BitBLT;
- ROP specialization;
- framebuffer updates;
- triangle rasterization.

The next low-risk graphics work should prefer:

- specializing common ROPs before changing generic `MIX()`;
- optimizing linear copy/fill cases before complex clipping/pattern paths;
- keeping the existing scalar implementation for all unsupported cases.

### 5. Future optional paths

Only after scalar-path equivalence and measurement:

- AVX2/SSE2/NEON specialization for clearly bounded video/memory paths;
- more aggressive dynarec translation;
- block linking;
- later, evaluation of a separate Vulkan-oriented design for compatible 3D batches.

## Benchmark instructions

### Instrumented benchmark build

```sh
cmake -S . -B build-bench \
  -G Ninja \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DPCEM_PERF_STATS=ON
cmake --build build-bench -j
```

Run:

```sh
./build-bench/src/pcem
```

### Benchmark method

For each run:

1. use the same machine, ROM set, CPU, RAM, storage, and video card;
2. boot the same guest;
3. run the same workload for the same time window;
4. close PCem cleanly;
5. save the generated `pclog`.

Compare logs between revisions, not just subjective guest responsiveness.

### Suggested workloads

Pick workloads that stress the area being measured:

- CPU-heavy DOS loops for dynarec validation;
- Windows 9x desktop and GUI operations for BitBLT;
- ViRGE-enabled demos or games for 2D/3D video paths.

### Benchmark limitations

- This repository does not currently ship a portable automated benchmark harness.
- ROMs and guest workloads are external.
- Results must be reported as real measurements from a known setup.

## Production build instructions

For a normal production-oriented build, leave perf counters disabled:

```sh
cmake -S . -B build-prod \
  -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DPCEM_PERF_STATS=OFF
cmake --build build-prod -j
```

Run:

```sh
./build-prod/src/pcem
```

## Production recommendations

- use `Release` unless you are actively debugging;
- keep `PCEM_PERF_STATS=OFF` for normal use;
- keep the default fallback execution paths enabled;
- only enable PGO after verifying a stable profile workflow for your toolchain;
- do not enable experimental changes and performance counters together when comparing release behavior.

## What remains after this document

After this README is added, the remaining work is functional, not documentation:

- collect benchmark logs from real guest workloads;
- decide the next optimization target from measured counters;
- implement one small reversible optimization at a time;
- re-measure after every change.
