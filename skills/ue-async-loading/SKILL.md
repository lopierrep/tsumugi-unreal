---
name: ue-async-loading
description: Patrón de carga asíncrona en UE5 — TSoftObjectPtr / TSoftClassPtr para refs que NO se cargan en memoria automáticamente, FStreamableManager + FStreamableHandle para async load (RequestAsyncLoad), patrones de scope (manage handle), level streaming (LoadStreamLevel), pitfalls (sync-load en hot path congela el frame, handle out-of-scope descarga, redirector chains rompen async load).
---

# UE Async Asset Loading

Patrón para cargar assets sin freezar el frame. Crítico en proyectos con muchos assets (open world, large character roster, modular weapons, etc.) o cualquier proyecto que abre niveles con muchas refs.

## Hard pointer vs Soft pointer

| Tipo | Carga | Cuándo usar |
|---|---|---|
| `UTexture2D*` / `TObjectPtr<UTexture2D>` | hard — el asset se carga AUTOMÁTICAMENTE al cargar el actor/objeto que lo referencia | Solo si el asset SIEMPRE se necesita inmediatamente al instanciar |
| `TSoftObjectPtr<UTexture2D>` | soft — el path se guarda, pero el asset NO se carga hasta que vos hacés explicit | Asset opcional, condicional, o que cargás just-in-time |
| `TSoftClassPtr<AMyActor>` | soft a una clase (no a una instancia) | Spawn condicional de actores |

**Regla mental**: si tu Blueprint/UCLASS referencia 1000 texturas vía hard pointers, al cargar el BP se cargan TODAS. Soft pointers rompen esa cadena.

## Soft pointer básico

```cpp
UCLASS()
class AMyWeapon : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere)
    TSoftObjectPtr<USoundBase> FireSound;

    void Fire();
};

void AMyWeapon::Fire()
{
    // Sync-load (BLOQUEA el frame): solo si el asset es chico y crítico AHORA
    USoundBase* Sound = FireSound.LoadSynchronous();
    if (Sound) UGameplayStatics::PlaySoundAtLocation(this, Sound, GetActorLocation());
}
```

**LoadSynchronous bloquea el frame**. Sólo aceptable para assets chiquitos (sonido corto, textura pequeña) Y solo cuando no hay alternativa async.

## Async load con FStreamableManager

Para assets grandes o cargas que no son urgentes (próximo frame está bien):

```cpp
#include "Engine/AssetManager.h"

void AMyWeapon::FireAsync()
{
    if (FireSound.IsPending())
    {
        UAssetManager& AM = UAssetManager::Get();
        FStreamableManager& Streamable = AM.GetStreamableManager();

        // Guardá el handle como member para que no se libere
        StreamHandle = Streamable.RequestAsyncLoad(
            FireSound.ToSoftObjectPath(),
            FStreamableDelegate::CreateUObject(this, &AMyWeapon::OnFireSoundLoaded),
            FStreamableManager::DefaultAsyncLoadPriority
        );
    }
    else
    {
        // Ya está en memoria, usar directo
        OnFireSoundLoaded();
    }
}

void AMyWeapon::OnFireSoundLoaded()
{
    USoundBase* Sound = FireSound.Get();
    if (Sound) UGameplayStatics::PlaySoundAtLocation(this, Sound, GetActorLocation());
}
```

**Header**:
```cpp
TSharedPtr<FStreamableHandle> StreamHandle;
```

## Pitfall: handle fuera de scope

```cpp
// BUG: Handle local sale de scope → la carga se cancela / asset descarga al terminar
void AMyWeapon::FireAsync()
{
    auto Handle = Streamable.RequestAsyncLoad(...);  // local!
}  // Handle se destruye acá, el async load se cancela
```

**Fix**: guardar el handle como member del actor (`TSharedPtr<FStreamableHandle> StreamHandle;`) o registrarlo con `ManageActiveHandle()` en el AssetManager si querés que viva un tiempo más.

## Cargar múltiples assets a la vez

```cpp
TArray<FSoftObjectPath> AssetsToLoad;
for (const TSoftObjectPtr<USoundBase>& Sound : SoundList)
{
    AssetsToLoad.Add(Sound.ToSoftObjectPath());
}

StreamHandle = Streamable.RequestAsyncLoad(
    AssetsToLoad,
    FStreamableDelegate::CreateUObject(this, &AMyWeapon::OnAllSoundsLoaded)
);
```

El callback se dispara una sola vez cuando TODOS los assets terminaron de cargar.

## Soft Class pointers (spawn condicional)

```cpp
UPROPERTY(EditAnywhere)
TSoftClassPtr<AMyEnemy> EnemyClass;

void AMyGameMode::SpawnEnemyAsync()
{
    UAssetManager::GetStreamableManager().RequestAsyncLoad(
        EnemyClass.ToSoftObjectPath(),
        FStreamableDelegate::CreateUObject(this, &AMyGameMode::DoSpawn)
    );
}

void AMyGameMode::DoSpawn()
{
    if (UClass* LoadedClass = EnemyClass.Get())
    {
        GetWorld()->SpawnActor<AMyEnemy>(LoadedClass, SpawnTransform);
    }
}
```

## Level streaming

Para cargar/descargar niveles enteros (sublevels o World Partition cells):

```cpp
FLatentActionInfo LatentInfo;
LatentInfo.CallbackTarget = this;
LatentInfo.ExecutionFunction = "OnLevelLoaded";
LatentInfo.UUID = 1;
LatentInfo.Linkage = 0;

UGameplayStatics::LoadStreamLevel(
    this,
    "MyLevel",
    /*bMakeVisibleAfterLoad=*/true,
    /*bShouldBlockOnLoad=*/false,
    LatentInfo
);
```

Con `bShouldBlockOnLoad=false`: load async, callback se dispara cuando termina. Con `true`: blocking load. Casi siempre querés false.

## World Partition

World Partition (UE5) maneja el streaming automáticamente por celdas espaciales. Vos no llamás load/unload manualmente. Pero:
- Sigue siendo importante usar soft references para refs de actor que no necesitás cargar siempre.
- Data Layers para state global (ciclo día/noche, eventos) con sus propios load streams.
- `WP Cell` size es crítico para perf: celdas chicas = mucho streaming churn; celdas grandes = mucha memoria.

## Pitfalls comunes

### 1. Sync-load en hot path

```cpp
void Tick(float DeltaTime)
{
    USoundBase* S = FireSound.LoadSynchronous();  // BLOQUEA cada tick
}
```

**Fix**: pre-cargar antes (en BeginPlay) o usar handle persistente.

### 2. Handle out-of-scope (ya cubierto arriba)

### 3. Redirector chains

Si moviste un asset (Move To... en editor), UE deja un redirector. `RequestAsyncLoad` sigue el redirector — costoso. Solución: **Fix Up Redirectors** en el folder (right-click → Fix Up Redirectors) periódicamente.

### 4. Asumiendo Get() no-null

`SoftPtr.Get()` retorna null si el asset NO está cargado. Siempre validar:

```cpp
if (USoundBase* S = FireSound.Get()) { ... }
// o
USoundBase* S = FireSound.LoadSynchronous();  // garantiza no-null si el path es válido
```

### 5. Cargar primary asset registry

El Asset Manager mantiene un registry. Si tu pipeline incluye scanning at startup, considerá `UAssetManager::Get().ScanPathsForPrimaryAssets(...)` para indexar primero, después acceder por PrimaryAssetId (más rápido que paths).

### 6. AsyncLoadPriority ignorado

Los priorities (Low/Default/High) influyen el ORDEN pero no el tiempo total. No es "High = inmediato". Para inmediato, sync-load. Para fondo, Low.

## What this skill is NOT for

- Implementar Asset Manager custom (cada proyecto AAA tiene el suyo, no es un patrón).
- Bake/cook pipeline (otra área).
- Streaming de meshes via Nanite (auto-managed por UE5).
- Hot reloading de assets en runtime (otra cosa, no streaming).
