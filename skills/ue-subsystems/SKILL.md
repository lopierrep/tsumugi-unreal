---
name: ue-subsystems
description: Patrón Subsystem en UE5 — reemplazo moderno de singletons, manager actors y GameMode-bloat. Conoce los 4 tipos (GameInstance, World, LocalPlayer, Engine), cuál usar según lifetime requerido, cómo acceder desde BP/C++, y los pitfalls comunes (no usar para state-per-PIE, no confundir con Components). Aplica a managers de audio, save game, inventory global, networking helpers, analytics, telemetry.
---

# UE Subsystem Pattern

Subsystems son el reemplazo moderno (UE 4.22+) para singletons, manager-actors y la práctica de tirar todo dentro de `GameMode`. Permite código modular con lifetime predecible, instanciación automática, y acceso global tipado.

## Los 4 tipos y cuándo usar cada uno

| Subsystem | Lifetime | Cuándo usar |
|---|---|---|
| `UEngineSubsystem` | Engine completo (1 sola instancia mientras corre el editor o el game) | Servicios cross-game: licensing, analytics globales, plugin systems |
| `UGameInstanceSubsystem` | GameInstance (vive entre levels, no entre PIE sessions) | Save game manager, audio manager, login session, party / matchmaking, achievement tracker |
| `UWorldSubsystem` | World (recrea con cada level / PIE / server travel) | World-scoped state: spatial grid, weather, day/night cycle, world clock, AI director |
| `ULocalPlayerSubsystem` | LocalPlayer (cada split-screen player tiene la suya) | UI state per player, input config per player, inventory local, settings per player |

## Pitfall: ¿GameInstance o World subsystem?

| Persiste entre levels | Persiste entre PIE | Subsystem |
|---|---|---|
| sí | sí (entre game sessions del editor) | `UGameInstanceSubsystem` |
| sí | no (cada PIE arranca limpio) | `UGameInstanceSubsystem` igual — PIE es un GameInstance fresh por sesión |
| no | no | `UWorldSubsystem` |

**Regla**: si necesitás que el state sobreviva a un `OpenLevel`, es GameInstance. Si no, es World.

## Cómo declarar (C++)

```cpp
// MyAudioSubsystem.h
#pragma once
#include "CoreMinimal.h"
#include "Subsystems/GameInstanceSubsystem.h"
#include "MyAudioSubsystem.generated.h"

UCLASS()
class MYGAME_API UMyAudioSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    UFUNCTION(BlueprintCallable)
    void PlayMusic(USoundBase* Music);

private:
    UPROPERTY()
    TObjectPtr<UAudioComponent> CurrentMusic;
};
```

```cpp
// MyAudioSubsystem.cpp
void UMyAudioSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    // setup
}

void UMyAudioSubsystem::Deinitialize()
{
    // teardown
    Super::Deinitialize();
}
```

UE registra automáticamente todos los `USubsystem` subclases en startup. **No tenés que instanciar nada vos**.

## Cómo acceder

**C++:**
```cpp
// Desde cualquier UObject con World
UWorld* World = GetWorld();
UMyWorldSubsystem* MySystem = World->GetSubsystem<UMyWorldSubsystem>();

// GameInstance
UGameInstance* GI = GetGameInstance();  // o World->GetGameInstance()
UMyAudioSubsystem* Audio = GI->GetSubsystem<UMyAudioSubsystem>();

// LocalPlayer
ULocalPlayer* LP = PlayerController->GetLocalPlayer();
UMyUISubsystem* UI = LP->GetSubsystem<UMyUISubsystem>();

// Engine (raro, pero válido)
UMyEngineSubsystem* Engine = GEngine->GetEngineSubsystem<UMyEngineSubsystem>();
```

**Blueprint:** todos los Subsystems se exponen como `Get <Subsystem Name>` automáticamente en el nodo Blueprint. No necesitás nodos custom.

## Inicialización condicional

Si tu subsystem no debe existir en ciertos contextos (e.g., servidor dedicado, editor-only), override `ShouldCreateSubsystem`:

```cpp
virtual bool ShouldCreateSubsystem(UObject* Outer) const override
{
    // Solo en cliente, no en servidor dedicado
    if (Cast<UGameInstance>(Outer)->IsDedicatedServerInstance())
        return false;
    return Super::ShouldCreateSubsystem(Outer);
}
```

## Pitfalls comunes

- **Confundir con `UActorComponent`**: Subsystems son globales (1 por scope). Components viven en un Actor concreto. Si pensaste "tiro un component que es manager global" → es un subsystem.
- **Usar `UEngineSubsystem` para game state**: vive entre game sessions y entre maps — terminás con state stale. Para casi todo es `UGameInstanceSubsystem` o `UWorldSubsystem`.
- **No considerar PIE**: si tirás cosas en `UEngineSubsystem`, cada PIE comparte el mismo subsystem → state leaks entre sesiones del editor. Casi siempre querés GameInstance.
- **Asumir que se inicializa antes que Actors**: el orden es: Engine → GameInstance → World (con sus subsystems) → Actors. No es legal acceder a actors en `Initialize()` del subsystem.
- **Replicar state en subsystems**: los subsystems no son `AActor`, no se replican. Para state networked usá un GameState subsystem helper + actor proxy, o componentes en GameState.

## Cuándo todavía tiene sentido un manager actor

- Necesitás que se replique → Actor (en `AGameStateBase` típicamente).
- Necesitás un tick controlado por la World tick group → Actor.
- Necesitás transform en mundo (e.g., AI director con visualización) → Actor.

## What this skill is NOT for

- Implementar un sistema concreto (audio manager, save manager) — solo enseña EL PATRÓN.
- Decidir entre Subsystem y Component — eso es otra skill (`ue-actor-component-vs-subsystem`).
- Network replication strategies — eso es `ue-replication-roles`.
