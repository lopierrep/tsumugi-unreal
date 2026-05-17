---
name: ue-scene-capture
description: Patrón de Scene Capture en UE5 — renderizar la escena desde puntos de vista adicionales (minimap, mirror, security camera, dataset capture, multi-view). Conocé los trade-offs (cada SceneCapture es un render pass entero, costo lineal con N), las ShowFlags seguras de desactivar para perf, y las anti-palancas cuando estás generando un dataset visual (no tocar shadows en RGB, mantener Tonemapper). Aplica a SceneCapture2D, SceneCaptureCubemap, planar reflections y cualquier setup con N cámaras simultáneas.
---

# UE Scene Capture Pattern

Patrón para renderizar la escena desde puntos de vista adicionales al main viewport. Aplica a múltiples casos: minimap top-down, mirror/portal rendering, security cameras, captura de dataset visual para ML, multi-view rendering, planar reflections.

## Cuándo aplicar este patrón

- Necesitás una vista de la escena que NO es la cámara principal.
- El número de cámaras es chico (1-20). Más allá hay que considerar alternativas.
- El target output es una textura (RenderTarget2D) que se consume en material, UI, o se exporta.

## Cuándo NO aplicar

- Solo querés un mirror plano y simple → `Planar Reflection Component` es más barato.
- Querés post-process effects sobre la vista principal → eso es `Post Process Volume` + custom material.
- Necesitás 1000+ vistas simultáneas (e.g. cámara por enemigo) → considerar render por demanda, atlas, o no-rendering approach.

## Hard facts: costo

Cada `SceneCapture2D` activo dispara un **render pass completo** cada frame (o cada N frames si seteás `bCaptureEveryFrame=false` + tickear vos mismo). Costo lineal con número de capturas.

- Game thread: bajo (solo enqueue).
- Render thread: medio-alto (otro pass entero, depthbuffer + GBuffer + lighting + post-process).
- GPU: alto (especialmente lighting + post-process si las flags no están tuneadas).
- Memoria: cada RenderTarget2D pesa `width × height × bytes_per_pixel` bytes en VRAM.

## ShowFlags: tabla de impacto

| Flag | Default | Safe to disable | Notas |
|---|---|---|---|
| `SetBloom(false)` | ON | sí | Costo medio-alto post-process |
| `SetLensFlares(false)` | ON | sí | Cosmético, suma con N cámaras |
| `SetMotionBlur(false)` | ON | sí | Pesado, especialmente para captura estática |
| `SetScreenSpaceReflections(false)` | ON | sí | Caro, especialmente multi-camera |
| `SetScreenSpaceAO(false)` | ON | sí | Caro en GBuffer-heavy scenes |
| `SetVolumetricFog(false)` | ON | sí | Muy caro con luces dinámicas |
| `SetDecals(false)` | ON | sí | Depende del proyecto |
| `SetTemporalAA(false)` | ON | sí (a veces) | TAA puede causar ghosting en captura — preferir FXAA o no-AA para dataset |
| `SetDynamicShadows(false)` | ON | **depende** | NO desactivar en captura RGB para dataset visual. Sí en captura para depth-only / thermal |
| `SetTonemapper(false)` | ON | **no** | Rompe color grading |
| `SetToneCurve(false)` | ON | **no** | Rompe color grading consistente |

Setter típico:
```cpp
SceneCapture->ShowFlags.SetBloom(false);
SceneCapture->ShowFlags.SetLensFlares(false);
SceneCapture->ShowFlags.SetMotionBlur(false);
SceneCapture->ShowFlags.SetScreenSpaceReflections(false);
SceneCapture->ShowFlags.SetScreenSpaceAO(false);
SceneCapture->ShowFlags.SetVolumetricFog(false);
SceneCapture->ShowFlags.SetDecals(false);
// IMPORTANT: NO tocar DynamicShadows ni Tonemapper si capturás dataset visual
```

## Anti-palancas cuando capturás dataset visual

Si el SceneCapture genera frames que van a entrenar un modelo / publicarse / consumirse como verdad visual, **rechazá** las siguientes "optimizaciones":

- Desactivar `DynamicShadows` en captura RGB — cambia la apariencia, rompe la asunción del dataset.
- Bajar resolución de SceneCapture — cambia la geometría de píxeles, invalida bounding boxes / segmentation.
- Cambiar `Tonemapper` o `ToneCurve` — rompe color grading consistente entre frames.
- Bajar `max view distance` / streaming range bajo ~200m — produce "wall-of-fog" artifact visible.

Si la perf es insuficiente con flags óptimas, las palancas correctas son:
1. **HLOD bake** con LOD0 cargando solo el sector activo.
2. **Nanite** en mallas pesadas (GPU moderna requerida).
3. **Capture interval** — bajar target FPS por captura si el dataset tolera tasa variable.

## Patrón: async readback para evitar GPU stalls

Si necesitás copiar el RenderTarget a CPU (e.g., para enviar por red, escribir a disco), hacelo **async**:
- `FRHIGPUTextureReadback` con N slots (típicamente 2) en pipeline.
- State machine: Empty → CaptureEnqueued → GpuStaging → Ready.
- Game thread hace broadcast cuando un slot llega a Ready; libera el slot.
- Render thread mueve los slots por las fases.

Sin async, el `Lock/Unlock` bloquea el game thread hasta que el GPU termine → frame time grande.

(Skill futura: `ue-async-gpu-readback` con el state machine detallado.)

## Alternativas a Scene Capture

| Necesidad | Alternativa más barata |
|---|---|
| Mirror plano simple | `Planar Reflection Component` |
| Cubemap estático | Bake offline, no real-time |
| 100+ cámaras (security walls) | Atlas grande + render con `Scene Depth` material |
| Render solo cuando se mira | `bCaptureEveryFrame=false` + tick por demanda |
| Render a baja tasa | `CaptureSortingPriority` + custom tick interval |

## Bundled references

- `references/showflags-table.md` — versión expandida de la tabla de arriba con notas adicionales (por crear).
- `references/scenecapture-vs-alternatives.md` — comparación detallada con planar reflections y cubemap baking (por crear).

## What this skill is NOT for

- Diagnóstico de FPS bajo en un proyecto específico → eso es trabajo de Claude usando esta skill + el `CLAUDE.md` del proyecto.
- Implementar async readback paso a paso → eso es skill futura.
- Optimización CPU game-thread → otro dominio.
