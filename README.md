# tsumugi-unreal

Plugin de la familia [tsumugi](https://github.com/lopierrep/tsumugi) con skills para proyectos de Unreal Engine 5.

## Skills incluidas

| Skill | Cuándo se dispara |
|---|---|
| `ue-perf-multi-camera` | El usuario reporta FPS bajo con varias SceneCapture simultáneas (multi-view rendering, datasets, drone sims). |
| `ue-sensor-readback` | El usuario reporta jitter, frames perdidos o sync mala entre sensores con async GPU readback. |
| `ue-vulkan-docker` | El usuario tiene problemas corriendo UE5 dentro de un Docker container Linux (libGLX, ICD manifest, CDI). |
| `ue-ned-spawn` | El usuario configura spawn de actores que se posicionan vía NED coordinates + terrain probing (drone sims, SITL). |

## Instalación

```shell
# Vía el orquestador tsumugi
/tsumugi:skills-install tsumugi-unreal

# O directo
/plugin marketplace add lopierrep/tsumugi
/plugin install tsumugi-unreal@tsumugi
```

## Filosofía

Las skills son **patrón-céntricas, no proyecto-céntricas**. Conocen los conceptos de UE5 (ShowFlags, SceneCapture, FRHIGPUTextureReadback, NED, etc.) pero los paths concretos del proyecto los lee Claude del `CLAUDE.md` del proyecto donde el usuario esté trabajando. Eso permite que la misma skill sirva para SenekaDrones, otro UE5 sim o un proyecto nuevo.

## Licencia

MIT.
