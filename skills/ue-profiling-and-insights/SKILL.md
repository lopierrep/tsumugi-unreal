---
name: ue-profiling-and-insights
description: Use whenever the user is debugging performance — looking for FPS drops, lag spikes, memory issues, render bottlenecks, or wants to add profiling markers to UE5 code. Triggers on phrases like "por qué va lento", "profilear esto", "Unreal Insights cómo", "stat unit", "GPU bottleneck UE", "cpu profile", "marcadores Insights", "profile this in UE5", "Unreal Insights setup", "stat commands UE", "find bottleneck", "TRACE_CPUPROFILER", "GPU vs CPU bound", "memreport". Returns the top-down profiling tree (stat commands → drill down → Insights), custom trace markers, GPU profiler usage, and memory tracking. UE5.4+ (notes deprecation of legacy stat profiler in 5.5+).
---

# UE Profiling & Insights

Sim/drone projects = many actors + physics + replication. Saber qué medir y cómo te ahorra horas. La key es **top-down**: empezar amplio, drill down.

## Top-down workflow

```
1. stat unit       (qué thread es el cuello: Game / Draw / GPU / RHIT)
   ↓
2. stat by domain  (drill al thread cuello)
   ↓
3. Unreal Insights (capture detallada con trace events)
   ↓
4. Custom markers  (TRACE_CPUPROFILER_EVENT_SCOPE en tu código)
```

## Stat commands (in-editor / runtime)

### Stat unit — primer paso, SIEMPRE

`stat unit` muestra los tiempos por frame para los 4 threads principales:

```
Frame: 16.7ms        (60 FPS budget)
Game:  3.2ms         (game thread — gameplay, AI, animation update)
Draw:  4.1ms         (render thread — preparing draw calls)
GPU:   9.5ms         (GPU — rendering)
RHIT:  1.2ms         (RHI thread — submitting to GPU driver)
```

Worst metric = bottleneck thread. Si Game está alto, drill al game thread. Si GPU está alto, drill al render.

### Drill al thread cuello

| Thread cuello | Stat command a usar | Qué muestra |
|---|---|---|
| Game thread | `stat game`, `stat tickables`, `stat anim`, `stat physics`, `stat ai` | Breakdown del game tick |
| Render / Draw | `stat scenerendering`, `stat lights`, `stat shadowrendering` | Costo de preparar drawcalls |
| GPU | `stat gpu`, `ProfileGPU` (frame-by-frame snapshot) | GPU passes y su costo |
| RHIT | raro | Driver / API overhead — usually fix is reduce drawcall count |

### Stat commands comunes (memo)

```
stat unit              # tiempos por thread (siempre primero)
stat fps               # FPS counter
stat game              # game thread breakdown
stat tickables         # qué tickables están costando
stat anim              # anim system cost
stat physics           # physics step cost
stat gpu               # GPU passes cost
stat scenerendering    # render pipeline breakdown
stat memory            # memory usage breakdown
stat startfile         # comenzar captura a archivo
stat stopfile          # parar captura → .uestats
ProfileGPU             # snapshot único GPU del frame siguiente
```

### Memory

```
memreport -full        # dump completo al archivo (logs/)
obj list               # lista UObjects activos
obj listmaps           # mapas cargados
stat memory            # categories de allocator
```

## Unreal Insights — captura detallada

### Setup

Launch UE con trace channels:

```
UnrealEditor.exe MyProject.uproject -trace=cpu,gpu,frame,bookmark,loadtime,counters,log
```

O en runtime:

```
Trace.Start cpu,gpu,frame
... do stuff ...
Trace.Stop
```

Trace file aparece en `<Project>/Saved/UnrealInsights/`. Abrir con `UnrealInsights.exe` (en `Engine/Binaries/Win64`).

### Channels más útiles

| Channel | Qué traza |
|---|---|
| `cpu` | Function-level CPU time (custom markers también) |
| `gpu` | GPU passes |
| `frame` | Frame boundaries |
| `bookmark` | Manual bookmarks (TRACE_BOOKMARK) |
| `loadtime` | Async load events |
| `counters` | Custom counters (TRACE_COUNTER) |
| `log` | UE_LOG output |
| `memtag` | Memory tags (LLM) |

### Custom markers

```cpp
#include "ProfilingDebugging/CpuProfilerTrace.h"

void UMyComponent::Tick(float Dt)
{
    TRACE_CPUPROFILER_EVENT_SCOPE(MyComponent_Tick);     // un scope, aparece en Insights timeline
    
    {
        TRACE_CPUPROFILER_EVENT_SCOPE(MyComponent_UpdateAI);
        UpdateAI();
    }
    {
        TRACE_CPUPROFILER_EVENT_SCOPE(MyComponent_UpdateAnim);
        UpdateAnimation();
    }
}
```

En Insights, vas a ver `MyComponent_Tick` con sub-scopes `MyComponent_UpdateAI` y `MyComponent_UpdateAnim` con sus tiempos.

### Counters

```cpp
TRACE_COUNTER_SET(TEXT("ActiveDrones"), DroneList.Num());
TRACE_COUNTER_INCREMENT(TEXT("FramesSinceLastSync"));
```

Aparecen como timelines de counter en Insights.

### Bookmarks

```cpp
TRACE_BOOKMARK(TEXT("Wave_%d_Started"), WaveNumber);
```

Marca puntos del tiempo en Insights para correlacionar.

## GPU profiling

### Stat gpu

```
stat gpu
```

Muestra pases (Base Pass, Lighting, Post-Process, etc.) con tiempos. Cuando algún pase sobresale, drill ahí.

### ProfileGPU

Snapshot del frame siguiente con desglose ultra-detallado:

```
ProfileGPU
```

Lista cada drawcall, su shader, su material, su mesh — todo cronometrado. Bueno para encontrar "un drawcall específico cuesta 5ms".

### GPU Visualizer

`ProfileGPU` + click en "GPU Visualizer" en el log = vista tree-map de costos por pasada y por mesh.

## Memory tracking

### Low-Level Memory (LLM)

Launch con `-llm`:

```
UnrealEditor.exe MyProject.uproject -llm
```

Activa tagging de allocations. En runtime:

```
stat llm           # breakdown por categoría
LLM.LogStats       # log al disk
```

Memreport completo:

```
memreport -full
```

→ `<Project>/Saved/Profiling/MemReports/MemReport-*.memreport`

## Pitfalls comunes

### 1. Profiling con editor running

Editor consume CPU/GPU significativo. Profiling debe ser:
- **Standalone Game** (no PIE) para perf real.
- **Development build** para tener stat commands habilitados.
- **Test/Shipping build** para perf de release (más optimizado).

### 2. PIE / single-window skews

Single-window PIE comparte threads con editor. Multi-window PIE separa pero igual no es Standalone. Para perf serio, usar Standalone Game (Play → Standalone Game).

### 3. Development vs Shipping

Development tiene stat commands y safe-checks habilitados. Shipping las quita. Los números difieren:
- Development: representa lo que veés mientras desarrollas.
- Shipping: lo que ven los usuarios. PUEDE ser 20-50% más rápido.

Profilá ambos cuando importe.

### 4. Stat profiler deprecado en 5.5+

El legacy "stat profiler" (UE4 era) está deprecado en 5.5+. Usar Insights ahora. Algunos tutorials viejos lo referían — ignoralos.

### 5. Insights archivos crecen rápido

Captura de 60s con full channels puede ser 100MB+. Usá `-trace=cpu,frame` mínimos si solo necesitás CPU.

### 6. Markers nesteados sin balance

```cpp
TRACE_CPUPROFILER_EVENT_SCOPE(A);
if (cond) return;   // ❌ scope A se cierra automáticamente — está OK
TRACE_CPUPROFILER_EVENT_SCOPE(B);   // ¿se cierra?
```

Los `_SCOPE` macros usan RAII — se cierran cuando salen del scope C++. Pero si vos manualmente abrís y cerrás, balanceá.

## Workflow

1. **Reproducí el escenario lento** en build apropiada (Standalone Development como default).
2. **Primer paso siempre**: `stat unit`. Identifica thread cuello.
3. **Drill al thread cuello** con stat commands específicas.
4. **Si necesitás más detalle**, capturá Insights con `-trace=cpu,gpu,frame`.
5. **Si el escenario es complejo**, agregá custom markers (TRACE_CPUPROFILER_EVENT_SCOPE) en el código sospechoso.
6. **Iterá**: cambio → re-medí → comparo.

## Output esperado

```
## Perf review — "frame drops cuando spawneo 100 drones"

### Diagnóstico
1. `stat unit` muestra Game thread 25ms (cuello).
2. `stat game` muestra `Drone_Tick` consumiendo 18ms — confirma.
3. Capturé Insights con `-trace=cpu,frame`. `Drone_Tick` tiene 2 hot spots: `UpdateAI` (12ms) y `UpdateSensors` (5ms).
4. Custom markers en `UpdateAI` → revela que `GetAllActorsOfClass(Threat)` adentro come 10ms (anti-pattern, ver `ue-anti-patterns`).

### Recomendación
Reemplazar `GetAllActorsOfClass` por subsystem registry (ver `ue-anti-patterns` #1). Estimo bajar `Drone_Tick` de 18ms → 4ms.

### Resumen
Bottleneck en game thread, no GPU. Anti-pattern `GetAllActorsOfClass` en tick. Fix conocido aplica.
```

## What this skill is NOT for

- Optimización específica (esa es trabajo aparte — esta skill DIAGNOSTICA).
- Profiling de tools / editor performance (UI lag, etc.).
- Network profiling (`stat net`, Network Profiler) — separado.
- Build performance (compile times, package times) — separado.
