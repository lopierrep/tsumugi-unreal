---
name: ue-debug-visualization
description: Use whenever the user needs to visualize runtime state in 3D — debug lines, spheres, paths, agent state, vector fields, sensor cones, AI behavior — in UE5. Triggers on phrases like "dibujar línea debug", "visualizar el path del dron", "Visual Logger UE", "GameplayDebugger nueva categoría", "overlay de debug", "DrawDebugSphere", "draw debug line", "visual logger UE5", "gameplay debugger category", "visualize agent state", "debug overlay", "UE_VLOG". Returns the toolkit: DrawDebug* functions, Visual Logger (timeline + 3D playback), GameplayDebugger (per-system overlay), on-screen messages, conditional debug macros. Especially valuable for sims (drones) where state is invisible.
---

# UE Debug Visualization

Sim / drones = state invisible (velocity vectors, target locks, sensor cones, pathfinding state). Sin visualización debuggeás ciegamente. Visual Logger especialmente es criminally underused.

## Las 4 herramientas

| Herramienta | Cuándo | Persistencia |
|---|---|---|
| `DrawDebug*` | Visualización inline, simple, on-demand | Single frame default; `bPersistentLines = true` opcional |
| Visual Logger | Logging + replay 3D con timeline scrubbing | Persistente entre runs |
| GameplayDebugger | Overlay categorizado de múltiples sistemas | Toggle en runtime con tecla apostrofe (`'`) |
| On-Screen Message | Text temporal en HUD | Configurable, expira |

## DrawDebug — inline visualization

`DrawDebug*` functions en `DrawDebugHelpers.h`. Disponibles en C++ y BP (vía `KismetSystemLibrary`).

### Las más usadas

```cpp
#include "DrawDebugHelpers.h"

// Una línea de A a B, color rojo, 3 segundos, grosor 2
DrawDebugLine(GetWorld(), A, B, FColor::Red, false, 3.f, 0, 2.f);

// Esfera en P, radio 50, color verde, persistente
DrawDebugSphere(GetWorld(), P, 50.f, 12, FColor::Green, true, 5.f);

// Box AABB
DrawDebugBox(GetWorld(), Center, Extent, FColor::Blue, false, 1.f);

// Cápsula (útil para character bounds)
DrawDebugCapsule(GetWorld(), Center, HalfHeight, Radius, Quat, FColor::Yellow);

// Texto floating en world
DrawDebugString(GetWorld(), Location, TEXT("Target Locked"), nullptr, FColor::White, 1.f);

// Flecha (vector display)
DrawDebugDirectionalArrow(GetWorld(), Start, End, 100.f, FColor::Cyan);

// Cono (sensor / FOV viz)
DrawDebugCone(GetWorld(), Origin, Direction, Length, AngleWidth, AngleHeight, 12, FColor::Magenta);

// Trayectoria parabólica
DrawDebugLine y DrawDebugSphere en loop sobre puntos de la trayectoria.
```

### Parameters comunes

| Param | Significado |
|---|---|
| `World` | `GetWorld()` |
| `bPersistentLines` | `false` = single frame, `true` = persiste hasta clear |
| `LifeTime` | Si `bPersistentLines = false`, segundos antes de desaparecer (0 = un frame) |
| `DepthPriority` | 0 = world (occluded por geometry), 100 = foreground (siempre visible) |
| `Thickness` | Grosor en pixels |

### Clear all debug

```cpp
FlushPersistentDebugLines(GetWorld());
```

### Patterns útiles

```cpp
// Visualizar velocidad como vector desde el actor
void Tick() {
    FVector V = GetVelocity();
    DrawDebugDirectionalArrow(GetWorld(), GetActorLocation(), 
        GetActorLocation() + V, 100.f, FColor::Yellow);
}

// Visualizar trayectoria predicha
TArray<FVector> Trajectory = PredictTrajectory(StartLocation, InitVelocity, Gravity, 60);
for (int i = 0; i < Trajectory.Num() - 1; i++) {
    DrawDebugLine(GetWorld(), Trajectory[i], Trajectory[i+1], FColor::Orange, false, 1.f);
}
```

### Pitfalls

- **Persistent leak**: si usás `bPersistentLines = true` sin clear, los debug shapes se acumulan. Llamá `FlushPersistentDebugLines` en BeginPlay o cuando cambies de escenario.
- **Hot loop slowdown**: 1000 `DrawDebugLine` por frame puede comer 3-5ms. Es debug — apagalo en shipping.

## Visual Logger — replay 3D con timeline

Visual Logger graba eventos + ubicaciones + texto a lo largo del tiempo. Después podés abrir el VisualLogger (Window → Developer Tools → Visual Logger) y **scrubbear** la timeline viendo qué pasaba en cada momento.

Es el equivalente "DVR" del debug.

### Setup

Declarar una categoría:

```cpp
#include "Logging/LogMacros.h"
#include "VisualLogger/VisualLogger.h"

DECLARE_LOG_CATEGORY_EXTERN(LogDroneNav, Log, All);
DEFINE_LOG_CATEGORY(LogDroneNav);
```

### Loggear eventos

```cpp
// Texto simple en el lifetime del actor
UE_VLOG(this, LogDroneNav, Log, TEXT("Drone reached waypoint %d"), WaypointIndex);

// Location marker en 3D
UE_VLOG_LOCATION(this, LogDroneNav, Log, GetActorLocation(), 50.f, FColor::Green, 
    TEXT("Waypoint reached"));

// Segmento (e.g., trayectoria)
UE_VLOG_SEGMENT(this, LogDroneNav, Log, OldLocation, GetActorLocation(), FColor::Blue, 
    TEXT("Moved"));

// Esfera
UE_VLOG_BOX(this, LogDroneNav, Log, FBox(Loc - FVector(50.f), Loc + FVector(50.f)), FColor::Red, 
    TEXT("Danger zone"));

// Histograma / counter
UE_VLOG_HISTOGRAM(this, LogDroneNav, Log, TEXT("HP"), TEXT("X"), FVector2D(GameTime, CurrentHP));
```

### Workflow

1. Add VLOG macros en código.
2. Play normalmente (PIE o Standalone).
3. Window → Developer Tools → Visual Logger.
4. Scrubea la timeline. Cada actor que loggeó aparece como track separado.
5. Click en eventos para ver texto + 3D viz a ese tiempo.

**Game-changer para sims**: ves la trayectoria del drone replayed, qué decisiones tomó, qué obstáculos detectó. Sin tener que reproducir el bug en vivo.

### Pitfall

- Visual Logger por default está OFF en cooked builds. Configurar `[VisualLogger]` en .ini para habilitarlo en dev cooked builds si necesitás.

## GameplayDebugger — overlay categorizado

Toggle con tecla apóstrofe (`'`). Aparece un overlay con categorías (NumPad 1-9 para activar/desactivar). Cada categoría es un sub-sistema (AI, Perception, Behavior Tree, Abilities, etc.).

### Categories built-in

- AI / Behavior Tree
- AI Perception
- Gameplay Abilities (si usás GAS)
- Pawn (basic info)
- Nav (navigation mesh)

### Tu categoría custom

```cpp
class FGameplayDebuggerCategory_Drone : public FGameplayDebuggerCategory
{
public:
    FGameplayDebuggerCategory_Drone();
    virtual void CollectData(APlayerController* OwnerPC, AActor* DebugActor) override;
    virtual void DrawData(APlayerController* OwnerPC, FGameplayDebuggerCanvasContext& CanvasContext) override;
    
    static TSharedRef<FGameplayDebuggerCategory> MakeInstance() { 
        return MakeShareable(new FGameplayDebuggerCategory_Drone()); 
    }
};
```

Registrar en module startup:

```cpp
IGameplayDebugger& GameplayDebuggerModule = IGameplayDebugger::Get();
GameplayDebuggerModule.RegisterCategory("Drone",
    IGameplayDebugger::FOnGetCategory::CreateStatic(&FGameplayDebuggerCategory_Drone::MakeInstance),
    EGameplayDebuggerCategoryState::EnabledInGameAndSimulate);
```

Ahora tu categoría aparece en el overlay. Override `CollectData` (recoge datos del actor) y `DrawData` (renderiza 2D HUD + 3D si querés).

## On-screen messages — quick & dirty

```cpp
GEngine->AddOnScreenDebugMessage(
    -1,                  // key (-1 = nuevo cada vez, otro número = reemplaza)
    5.f,                 // duración
    FColor::Yellow,
    FString::Printf(TEXT("HP: %.1f"), CurrentHP)
);
```

**Pitfall**: key = -1 spamea N mensajes. Usá key stable (e.g., `1001`) para que el mismo mensaje se reemplace:

```cpp
GEngine->AddOnScreenDebugMessage(1001, 5.f, FColor::White, FString::Printf(TEXT("HP: %.1f"), CurrentHP));
```

## Gating con `WITH_EDITOR` / cvars

Para que debug code NO ship a release:

```cpp
#if WITH_EDITOR
    DrawDebugLine(...);
#endif
```

O usar cvars (toggleable runtime):

```cpp
static TAutoConsoleVariable<int32> CVarShowDroneDebug(
    TEXT("Drone.ShowDebug"),
    0,
    TEXT("Show drone debug visualization. 0=off, 1=basic, 2=verbose"));

void Tick() {
    int32 Level = CVarShowDroneDebug.GetValueOnGameThread();
    if (Level >= 1) {
        DrawDebugDirectionalArrow(...);
    }
    if (Level >= 2) {
        // verbose viz
    }
}
```

En runtime: console `Drone.ShowDebug 1` (o 2 o 0).

## Workflow

1. **Identificá qué querés ver**: state interno de UN actor (DrawDebug en tick), comportamiento a lo largo del tiempo (Visual Logger), overlay multi-system (GameplayDebugger), valor escalar quick (on-screen message).
2. **Elegí la herramienta correcta** según la tabla.
3. **Gateá** con cvars o WITH_EDITOR para no shippear.
4. **Iterá**: rapido (DrawDebug) para casos puntuales, Visual Logger para sims complejas.

## What this skill is NOT for

- Profiling perf (eso es `ue-profiling-and-insights`).
- Logging textual sin viz 3D (eso es `ue-logging-and-assertions`).
- UI debugging (DevTools, UMG debug — separate concern).
- Network state visualization especifica (parcialmente cubierto por GameplayDebugger).
