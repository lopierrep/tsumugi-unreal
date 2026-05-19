---
name: ue-logging-and-assertions
description: Use whenever the user is configuring logging in UE5 — declaring log categories, choosing UE_LOG verbosity, deciding between check/ensure/verify, or fixing "my UE_LOG doesn't appear". Triggers on phrases like "UE_LOG no aparece", "categoría de log nueva", "check vs ensure", "logs en shipping", "verbosity UE", "assertions en UE", "declare log category", "UE_LOG vs ensure vs check", "logs not showing", "logging verbosity UE5", "assertion strategy unreal", "UE_LOGFMT". Returns the canonical setup for log categories, the verbosity ladder + which compiles out where, runtime controls, and the check/ensure/verify decision table.
---

# UE Logging & Assertions

Sim projects log heavy. Wrong category setup = log ruido. Wrong assertion = crashes en shipping o silent failures. La mayoría defaultea a `LogTemp` + `Warning` siempre — mala práctica.

## Log categories: SIEMPRE declarar la tuya

NO uses `LogTemp` para nada serio. `LogTemp` es para experimentos rapidos.

### Declarar una categoría

**Header** (uno por module, o uno por subsistema grande):
```cpp
// MyDroneNavLog.h
#pragma once
#include "Logging/LogMacros.h"

DECLARE_LOG_CATEGORY_EXTERN(LogDroneNav, Log, All);
```

**Source** (uno solo):
```cpp
// MyDroneNavLog.cpp
#include "MyDroneNavLog.h"

DEFINE_LOG_CATEGORY(LogDroneNav);
```

Macros:
- `DECLARE_LOG_CATEGORY_EXTERN(<Name>, <DefaultVerbosity>, <CompileTimeVerbosity>)`
- `DefaultVerbosity`: el level por default si nadie lo cambia en runtime. `Log` es típico.
- `CompileTimeVerbosity`: lo que se COMPILA en el binary. Mensajes por debajo se eliminan totalmente (perf-friendly). En Shipping muchos categorias usan `Warning` como compile-time → DEBUG/Verbose desaparecen.

### Uso

```cpp
UE_LOG(LogDroneNav, Log, TEXT("Drone %s reached waypoint %d"), *GetName(), WaypointIdx);
UE_LOG(LogDroneNav, Warning, TEXT("Drone %s lost signal"), *GetName());
UE_LOG(LogDroneNav, Error, TEXT("Drone %s emergency landing triggered"), *GetName());
```

### Por qué NO usar `LogTemp`

- Todo se mezcla con noise de UE engine logs.
- No podés filtrar por sistema.
- En equipo, nadie sabe qué messages son tuyos.

## Verbosity ladder

| Level | Cuándo |
|---|---|
| `Fatal` | El game va a crashear. `UE_LOG(..., Fatal, ...)` crashea inmediato. |
| `Error` | Algo falló pero el game sigue. Devs deberían ver esto. |
| `Warning` | Inusual / sub-óptimo. No rompe pero merece atención. |
| `Display` | Para messages que se MUESTRAN siempre en console (no filtrable normalmente). |
| `Log` | Info significativa del runtime. Default verbosity. |
| `Verbose` | Detalle para investigación. Off por default en shipping. |
| `VeryVerbose` | Extremo. Per-frame, per-tick. Off salvo durante deep debug. |

### Runtime control

En console:
```
log LogDroneNav VeryVerbose      # subir verbosity de esa categoría
log LogDroneNav off               # silenciar la categoría
log list                          # mostrar categorías y sus niveles actuales
```

En `Config/DefaultEngine.ini`:
```ini
[Core.Log]
LogDroneNav=Log
LogDroneNav.CompileTime=Verbose
```

## UE_LOGFMT — structured logging (5.x)

UE 5.x introduce `UE_LOGFMT` con structured fields (formato fmtlib + structured KVs):

```cpp
UE_LOGFMT(LogDroneNav, Log, "Drone {Name} at waypoint {WaypointIdx} with battery {Battery}%",
    ("Name", *GetName()),
    ("WaypointIdx", WaypointIdx),
    ("Battery", BatteryPct));
```

**Pro**: queryable structured fields (joinable en log aggregators).
**Cuándo**: si tu pipeline log post-procesa logs (Datadog, Elastic), preferí UE_LOGFMT. Si solo logs locales/console, UE_LOG tradicional está OK.

## Conditional log: UE_CLOG

```cpp
UE_CLOG(CurrentHP < 10, LogDroneNav, Warning, TEXT("Drone %s low HP"), *GetName());
```

Solo logea si la condición es true. **Útil para evitar formatting cost** cuando la condición es rare:

```cpp
// MAL: formatting siempre ocurre
UE_LOG(LogDroneNav, VeryVerbose, TEXT("State: %s"), *VerboseStateString());

// BIEN: si VeryVerbose está off, ni evalúa VerboseStateString()
UE_CLOG(UE_LOG_ACTIVE(LogDroneNav, VeryVerbose), LogDroneNav, VeryVerbose, 
    TEXT("State: %s"), *VerboseStateString());
```

## Assertions: check, ensure, verify

Decision table:

| Macro | Fatal? | Compila en Shipping? | Cuándo usar |
|---|---|---|---|
| `check(Expr)` | Sí, crash | NO (compila a no-op) | Invariantes que NUNCA deben fallar. Debug-only. |
| `checkSlow(Expr)` | Sí | NO en non-Debug builds | Invariantes en hot paths que solo querés chequear en Debug |
| `checkf(Expr, Fmt, ...)` | Sí | NO | Como `check` con mensaje formatted |
| `ensure(Expr)` | NO (warning + breakpoint) | SÍ (compila siempre) | Algo no debería pasar pero quiero seguir y avisarme. **Se dispara solo una vez por session por callsite.** |
| `ensureMsgf(Expr, Fmt, ...)` | NO | SÍ | Como `ensure` con mensaje |
| `ensureAlways(Expr)` | NO | SÍ | Como `ensure` pero se dispara cada vez (no once-per-session) |
| `verify(Expr)` | NO | SÍ (evalúa siempre) | Como `check` pero la expression se evalúa también en Shipping (sin chequeo de invariante) |

### Cuándo cada uno

**`check`**: invariantes internas que SI fallan, el código siguiente está roto.
```cpp
void ProcessOrder(const FOrder& Order) {
    check(Order.Items.Num() > 0);   // si está vacío, hay bug arriba, mejor crashear ahora
    // ... procesar
}
```

**`ensure`**: condiciones que no DEBERÍAN pasar pero quiero loggear y seguir.
```cpp
void OnEnemyKilled(AEnemy* E) {
    ensure(E != nullptr);   // null no debería llegar, pero si llega, log + return graceful
    if (!E) return;
    // ...
}
```

**Caveat de `ensure`**: SOLO SE DISPARA UNA VEZ por callsite por sesión. Bueno para evitar spam, malo si esperás verlo cada vez. Para "cada vez", usar `ensureAlways`.

**`verify`**: si la expression tiene side effects que necesitás incluso en Shipping:
```cpp
verify(SomeImportantInit() == EOK);   // SomeImportantInit() corre en Shipping también
```

(Raro de usar. Si lo necesitás, probablemente refactor.)

## Pitfalls comunes

### 1. UE_LOG no aparece

Posibles causas:
- Verbosity en runtime es más bajo que el level del UE_LOG. Fix: `log <Category> Verbose` en console.
- CompileTimeVerbosity stripped el message (eg., usás `Verbose` pero compile-time = `Warning`). Fix: ajustar declaración en `DECLARE_LOG_CATEGORY_EXTERN`.
- Estás corriendo en build sin logs (Shipping con minimal logs). Fix: cambiar a Development.

### 2. UE_LOG en hot path con formatting costoso

```cpp
void Tick() {
    UE_LOG(LogDrone, VeryVerbose, TEXT("State: %s"), *ComputeExpensiveStateString());
    // ❌ ComputeExpensiveStateString se llama SIEMPRE, aunque VeryVerbose esté off
}
```

**Fix**: `UE_CLOG` con guard:
```cpp
UE_CLOG(UE_LOG_ACTIVE(LogDrone, VeryVerbose), LogDrone, VeryVerbose, TEXT("State: %s"), 
    *ComputeExpensiveStateString());
```

### 3. Shipping Fatal asserts from designer-misconfig data

```cpp
check(WeaponConfig != nullptr);   // crashea en Shipping si designer olvidó settearlo
```

**Fix**: `ensureMsgf` para data-driven configs:
```cpp
ensureMsgf(WeaponConfig, TEXT("WeaponConfig is null — designer must set it"));
if (!WeaponConfig) return;   // graceful fallback
```

### 4. Swallowing errors con ensure y continuar

```cpp
ensure(SomeImportantCheck());   // warning si falla
// ... seguir como si nada
```

`ensure` NO bloquea. Es solo un report. Si el invariante MATTERS para el siguiente código, agregá un `if (!cond) return;` también.

### 5. Logs en BP

En Blueprints, `Print String` está OK para debug rápido pero NO usa categorías ni verbosity. Para BP en producción, usar `Log` node (lleva al log file con UE_LOG behavior, configurable). Print String es el equivalente "on-screen message".

## Workflow

1. **Identificá si es logging o assertion**:
   - Logging = "quiero ver qué pasó"
   - Assertion = "el código siguiente asume X; si X no se cumple, X es bug"

2. **Para logging**:
   - Declarar categoría propia (no LogTemp).
   - Elegir verbosity correcto (Warning para "fix me", Log para info normal, Verbose para investigation).
   - Si hay formatting costoso, UE_CLOG con guard.

3. **Para assertions**:
   - `check`: invariante interna, fail-fast.
   - `ensure`: anomalía esperable, log + continuar.
   - `verify`: solo si la expression tiene side effect crítico en Shipping.

4. **Validar Shipping**: ¿qué pasa con tus logs/asserts en Shipping build?

## What this skill is NOT for

- Visualización 3D (eso es `ue-debug-visualization`).
- Profiling (eso es `ue-profiling-and-insights`).
- Logging cross-service / aggregation (eso es infra, no UE-side).
- Replay / scrubbable logs en 3D (eso es Visual Logger, en `ue-debug-visualization`).
