---
name: ue-data-driven-design
description: Use whenever the user is deciding how to store configuration / parameters / stats in UE5 — DataAsset, PrimaryDataAsset (+ AssetManager), DataTable, CurveTable, DeveloperSettings, or UStruct defaults. Triggers on phrases like "DataAsset o DataTable", "datos de configuración de drones", "tabla de stats", "cargar assets por ID", "Primary Data Asset", "AssetManager UE", "drive config from data", "curve table for tuning", "data-driven UE5", "Lyra data structure". Returns a decision tree + when to use each + integration with AssetManager for async load by ID. Especially valuable for sim projects (drones, missions, fleets) where data layout drives everything.
---

# UE Data-Driven Design

Sim projects (drones, fleets, missions, RPG configs) viven y mueren por data layout. Elegir mal acá significa rewrites costosos. Lyra usa PrimaryDataAssets heavily por una razón.

## Decision tree

```
¿Qué necesitás representar?
│
├── Multiple rows of the SAME SHAPE (table)
│   ├── Designers necesitan editarlo en spreadsheet → DataTable (CSV import)
│   ├── Numeric curves para tuning (damage vs distance) → CurveTable
│   └── Static rows que cambian raramente → DataTable
│
├── Single configurable asset (one "thing" with rich data)
│   ├── No cargado en runtime automáticamente / queryable by ID → PrimaryDataAsset (con AssetManager)
│   ├── Cargado siempre cuando se usa, sin ID lookup → DataAsset
│
├── Global project settings (1 valor por proyecto, viewable en Project Settings)
│   └── DeveloperSettings
│
└── Lightweight value type embedded en otro asset
    └── UStruct con defaults
```

## Cada opción: cuándo y cómo

### `UDataAsset`

**Cuándo**: un "thing" configurable que vive como asset propio en el Content Browser. Ejemplo: `BP_WeaponConfig`, `DA_DroneStats`, `DA_MissionParameters`.

```cpp
UCLASS(BlueprintType)
class UDroneStats : public UDataAsset
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere)
    float MaxSpeed = 30.f;

    UPROPERTY(EditAnywhere)
    float MaxAltitude = 500.f;

    UPROPERTY(EditAnywhere)
    TSoftObjectPtr<UStaticMesh> Body;
};
```

**Uso**: el actor lo refie via UPROPERTY:
```cpp
UPROPERTY(EditDefaultsOnly)
TObjectPtr<UDroneStats> Stats;
```

Designer asigna en el editor. Hot-reloadable.

**Pro**: lightweight, no requiere AssetManager.
**Con**: no queryable por ID — necesitás hard ref desde donde se use.

### `UPrimaryDataAsset` + AssetManager

**Cuándo**: necesitás cargar por ID, async, sin hard refs. Ejemplo: catálogo de items en RPG, lista de missions, biblioteca de enemies. Lyra usa esto extensamente.

```cpp
UCLASS()
class UMissionDataAsset : public UPrimaryDataAsset
{
    GENERATED_BODY()
public:
    virtual FPrimaryAssetId GetPrimaryAssetId() const override
    {
        return FPrimaryAssetId(TEXT("Mission"), GetFName());
    }

    UPROPERTY(EditAnywhere)
    FText Title;
    
    UPROPERTY(EditAnywhere)
    TArray<TSoftObjectPtr<AActor>> RequiredActors;
};
```

**Setup**: en `DefaultGame.ini`:
```ini
[/Script/Engine.AssetManagerSettings]
+PrimaryAssetTypesToScan=(PrimaryAssetType="Mission",
    AssetBaseClass=/Script/MyGame.MissionDataAsset,
    bHasBlueprintClasses=False,
    bIsEditorOnly=False,
    Directories=(("/Game/Missions")),
    SpecificAssets=,
    Rules=(Priority=0,bApplyRecursively=True,CookRule=AlwaysCook))
```

**Uso runtime**:
```cpp
UAssetManager& AM = UAssetManager::Get();
TArray<FPrimaryAssetId> Ids;
AM.GetPrimaryAssetIdList(TEXT("Mission"), Ids);

// Async load por ID
FPrimaryAssetId Id = Ids[0];
TSharedPtr<FStreamableHandle> Handle = AM.LoadPrimaryAsset(Id, {}, 
    FStreamableDelegate::CreateUObject(this, &UMyManager::OnLoaded));
```

**Pro**: queryable by ID, async-load friendly, no hard refs.
**Con**: more setup ceremony.

### `UDataTable`

**Cuándo**: rows of the SAME SHAPE — stats por enemigo, items con price/damage/icon, niveles con goal/timer/bonus.

```cpp
USTRUCT(BlueprintType)
struct FEnemyStats : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere) int32 HP;
    UPROPERTY(EditAnywhere) float Speed;
    UPROPERTY(EditAnywhere) TSoftObjectPtr<USkeletalMesh> Mesh;
};
```

Crear DataTable asset con `FEnemyStats` como row type. Designers editan en editor o importan CSV.

**Uso**:
```cpp
UPROPERTY(EditDefaultsOnly)
TObjectPtr<UDataTable> EnemyTable;

const FEnemyStats* Stats = EnemyTable->FindRow<FEnemyStats>(FName("Goblin"), TEXT("Spawn"));
```

**Pro**: spreadsheet-friendly, designer-editable, fast lookup by FName.
**Con**: lookup by FName puede ser lento si llamado en hot loop (cachear).

### `UCurveTable`

**Cuándo**: curvas numéricas para tuning. "Damage falloff por distancia", "XP per level", "spawn rate por tiempo".

Similar a DataTable pero cada row es una curva (no un struct).

```cpp
FRealCurve* DmgCurve = MyCurveTable->FindCurve(FName("RifleDamage"), TEXT(""));
float Damage = DmgCurve->Eval(distanceMeters);
```

**Pro**: designers tweak curves visualmente en editor.

### `UDeveloperSettings`

**Cuándo**: project-wide config que aparece en Project Settings.

```cpp
UCLASS(Config = Game, defaultconfig, meta = (DisplayName = "Drone Sim Settings"))
class UDroneSimSettings : public UDeveloperSettings
{
    GENERATED_BODY()
public:
    UPROPERTY(Config, EditAnywhere)
    float DefaultMissionTime = 600.f;
};
```

Persists in `Config/DefaultGame.ini`. Accessible via:
```cpp
const UDroneSimSettings* S = GetDefault<UDroneSimSettings>();
float Time = S->DefaultMissionTime;
```

**Pro**: settings centralizados, persistencia automática en .ini.
**Con**: para values que cambian en runtime (no read-only), no es lo correcto — usar GameUserSettings.

### `UStruct` defaults

**Cuándo**: lightweight bundle de values que vive embedded en otro asset. NO necesita ser su propio asset.

```cpp
USTRUCT(BlueprintType)
struct FCameraConfig
{
    GENERATED_BODY()
    UPROPERTY(EditAnywhere) float FOV = 90.f;
    UPROPERTY(EditAnywhere) float NearClip = 10.f;
};

// En el actor o DataAsset:
UPROPERTY(EditAnywhere)
FCameraConfig CameraConfig;
```

## Patterns importantes

### Async load por PrimaryAssetId

```cpp
// Carga 1 asset por ID
UAssetManager::Get().LoadPrimaryAsset(
    FPrimaryAssetId("Mission", "Mission_01"),
    {},
    FStreamableDelegate::CreateUObject(this, &UMyManager::OnMissionLoaded)
);

// Carga lista de assets por type
TArray<FPrimaryAssetId> Ids;
UAssetManager::Get().GetPrimaryAssetIdList(TEXT("Mission"), Ids);
```

(Handoff a `ue-async-loading` para más detalle.)

### CSV import workflow

1. Crear `USTRUCT` con `FTableRowBase`.
2. Crear DataTable asset en editor (apuntando al struct).
3. Export del DataTable → CSV inicial.
4. Designer edita CSV en spreadsheet.
5. Re-import en editor.

### Lyra-style organization

Lyra usa PrimaryDataAsset para todo gameplay-configurable:
- `LyraExperienceDefinition` (define qué pasa en una sesión)
- `LyraGameplayAbilitySet` (set de abilities)
- `LyraInputConfig` (mappings de input)
- `LyraPawnData` (config del pawn)

Cada uno es PrimaryDataAsset cargado por AssetManager. Designers asignan via UI sin tocar código.

## Pitfalls comunes

### 1. DataTable lookup en hot loop

`FindRow<T>(FName, ...)` es O(1) average pero hace string-hash lookup. En tick loop con 1000 calls, cachear:

```cpp
// Cache once
const FEnemyStats* CachedGoblinStats = EnemyTable->FindRow<FEnemyStats>("Goblin", "");

void Tick() {
    UseStats(CachedGoblinStats);   // raw deref, fast
}
```

### 2. PrimaryDataAsset sin scan path

Si tu PrimaryDataAsset no aparece en `GetPrimaryAssetIdList`, falta el config en `DefaultGame.ini`. AssetManager solo conoce los types que le declaraste.

### 3. Hard ref a heavy asset en DataAsset

```cpp
class UCharacterStats : public UDataAsset
{
    UPROPERTY()
    TObjectPtr<UStaticMesh> Mesh;   // ❌ HARD ref → mesh cargado siempre que CharacterStats se carga
};
```

**Fix**: TSoftObjectPtr para assets pesados.

### 4. CurveTable vs DataTable con float column

Si solo necesitás un mapeo X → Y, usar CurveTable (designer ve la curva). Si necesitás múltiples columns relacionados, DataTable.

### 5. DeveloperSettings para runtime mutables

DeveloperSettings es para valores que se settean una vez. Si tu setting cambia en runtime (volumen del sonido, brightness), usar `UGameUserSettings` o un Save Game.

## Workflow

1. **Identificá el shape de los datos**: ¿N rows iguales? ¿1 thing configurable? ¿Curva? ¿Project setting? ¿Embedded en otro?
2. **Aplicá el decision tree**.
3. **Setup**: declarar struct/class + asset en editor + config (.ini si aplica).
4. **Validar**: refs son soft o hard según ownership. Cachear lookups si es hot path.
5. **Reportá** la decisión + ejemplo de código.

## What this skill is NOT for

- Async loading per-se (eso es `ue-async-loading`).
- Save game / persistence (UGameUserSettings, USaveGame).
- Localization (FText vs FString — separate concern).
- GAS configuration (eso usa PrimaryDataAsset pero es topic específico).
