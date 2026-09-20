# Phase 1: dynarec 386 / S3 ViRGE

Esta entrega mantiene la semántica existente y se limita a instrumentación opcional, desactivada por defecto.

## Arquitectura revisada

- `src/cpu/386_dynarec.c`
  - `exec_interpreter()` ejecuta bloques completos por `x86_opcodes[...]` y corta por cambio de página, `abrt`, `trap`, SMI/NMI o fin de bloque.
  - `exec_recompiler()` hace validación rápida por `codeblock_hash`, valida `pc/_cs/phys/status`, cae a búsqueda lenta `codeblock_tree_find()` cuando falla, comprueba `dirty_mask/page_mask`, y recompila o marca bloques según el estado.
  - Durante recompilación, cada instrucción sigue pasando por el handler clásico `x86_opcodes[...]` después de `codegen_generate_call(...)`.
- `includes/private/codegen/codegen.h`
  - `codeblock_t` conserva `pc`, `phys`, `status`, `flags`, máscaras por página (`page_mask`, `page_mask2`) y punteros a `dirty_mask`, además de listas por página y árbol de búsqueda.
- `src/codegen/codegen_block.c`
  - `codegen_check_flush()` invalida bloques cuando `dirty_mask & page_mask` intersectan.
  - `codegen_block_init()`, `codegen_block_start_recompile()`, `codegen_block_end()` y `codegen_block_end_recompile()` son los puntos seguros para medir creación/finalización de bloques.
- `src/codegen/codegen.c` y `src/codegen/codegen_x86-64.c`
  - `codegen_generate_call()` decide entre traductor nativo (`recomp_op_table[...]`) y fallback a handler C (`uop_CALL_INSTRUCTION_FUNC` o equivalente x86-64).
- `src/cpu/cpu.c`
  - `x86_setopcodes()` conecta tablas `x86_opcodes` y `x86_dynarec_opcodes`.
- `src/video/vid_s3_virge.c`
  - `s3_virge_bitblt()` concentra BitBLT, rectfill, line y poly.
  - `MIX()` aplica el ROP bit a bit.
  - `tri()` y `s3_virge_triangle()` llevan la rasterización 3D, acceso a VRAM y muestreo de texturas.

## Cuellos de botella y riesgos de compatibilidad

- El dynarec actual no puede asumirse “mayoritariamente nativo”: la decisión real entre traductor nativo y handler C ocurre dentro de `codegen_generate_call()` y depende de la tabla/opcode concretos.
- La validación de bloques depende de:
  - `pc/_cs/phys/status` actuales;
  - máscaras por página y páginas sucias;
  - recompilación por `TOP` estático de FPU.
  Por eso no se forzó todavía una caché de “último bloque” sin medir antes estos casos.
- La invalidación de código está acoplada a `mem_flush_write_page()`, `dirty_mask`, `code_present_mask` y a listas/árboles de `codeblock_t`; un atajo incorrecto puede romper self-modifying code.
- En ViRGE, BitBLT y triángulos mezclan clipping, direcciones invertidas, varios formatos y accesos directos a VRAM; optimizar `MIX()` o copiar memoria sin especialización explícita sería arriesgado.

## Instrumentación añadida

Compilar con:

```sh
cmake -S /home/runner/work/pcem/pcem -B /home/runner/work/pcem/pcem/build -DPCEM_PERF_STATS=ON
cmake --build /home/runner/work/pcem/pcem/build -j
```

Con `PCEM_PERF_STATS=ON`, al cerrar el emulador se vuelcan a `pclog`:

- Dynarec 386:
  - bloques interpretados;
  - entradas al recompiler;
  - validaciones rápidas correctas;
  - fallos de caché hash;
  - validaciones lentas y aciertos;
  - comprobaciones por páginas sucias;
  - recompilaciones por `TOP` de FPU;
  - bloques compilados, ejecutados y solo marcados;
  - invalidaciones;
  - excepciones;
  - instrucciones traducidas nativamente frente a fallbacks a handler C.
- S3 ViRGE:
  - operaciones y píxeles de BitBLT;
  - operaciones y píxeles de rectfill;
  - operaciones line/poly;
  - triángulos, píxeles rasterizados y muestras de textura;
  - lecturas/escrituras de VRAM observadas en estas rutas;
  - operaciones escalares;
  - tiempo acumulado en ticks;
  - histograma de ROP usado.

## Benchmarks base

No hay infraestructura de benchmark automatizada en este árbol ni ROMs/fixtures portables incluidos para ejecutar una línea base completa en el entorno del agente.

Comandos reproducibles propuestos (no ejecutados aquí):

```sh
cmake -S /home/runner/work/pcem/pcem -B /home/runner/work/pcem/pcem/build-rel -DCMAKE_BUILD_TYPE=RelWithDebInfo -DPCEM_PERF_STATS=ON
cmake --build /home/runner/work/pcem/pcem/build-rel -j
/home/runner/work/pcem/pcem/build-rel/src/pcem
```

Procedimiento manual sugerido (no ejecutado aquí):

1. Abrir una máquina 386 con dynarec activado y una máquina con S3 ViRGE.
2. Ejecutar la misma carga DOS/Windows 9x/juego durante una ventana fija.
3. Cerrar PCem y recoger el `pclog`.
4. Comparar:
   - `native_translated_instructions` vs `handler_calls`;
   - `cache_misses`, `slow_validations`, `block_invalidations`;
   - `bitblt_pixels`, `triangles`, `rasterized_pixels`, `rop[...]`.

## Limitaciones actuales

- No se añadió una optimización de caché de bloques ni AVX2 en esta fase porque la semántica de invalidación y las pruebas disponibles no permiten demostrar equivalencia exacta en este entorno.
- El porcentaje de handlers C solo es medible cuando `PCEM_PERF_STATS=ON` y se recompilan bloques durante una ejecución real.
- No se afirman mejoras de rendimiento en este documento.
