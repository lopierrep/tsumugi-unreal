---
name: ue-anti-patterns
description: Use whenever the user shows UE5 code (C++ or Blueprint) and is wondering if it follows best practices, or is debugging perf/coupling/loading issues that smell like anti-patterns. Triggers on phrases like "esto es buena práctica en UE", "cast hell", "está spammeando GetAllActors", "el blueprint está cargando todo", "Tick está pesado", "anti-pattern UE", "refactor de Cast", "is this a UE anti-pattern", "blueprint load chain", "decouple blueprints". Returns the canonical anti-pattern catalog + how to detect each + the modern UE5 fix (subsystems, gameplay tags, interfaces, soft refs, event-driven). Applies to UE5 + Blueprint.
---

# UE Anti-Patterns

Catálogo de los anti-patterns canónicos que funcionan en prototype y destruyen shipping projects. Cada uno tiene un replacement pattern moderno: GameplayTags, interfaces, subsystems, soft refs, composition.

## El catálogo

### 1. `GetAllActorsOfClass` en Tick (o frecuente)

```cpp
// ❌ ANTI-PATTERN
void Tick(float DeltaTime) {
    TArray<AActor*> Enemies;
    UGameplayStatics::GetAllActorsOfClass(this, AEnemy::StaticClass(), Enemies);
    // ... do something
}
```

**Por qué malo**: scan O(N) sobre TODOS los actors del world cada frame. En levels grandes (1000+ actors), esto es 60×1000 = 60k ops/segundo en una skill que existe N veces.

**Fix moderno**: 
- **Registry pattern via Subsystem**: actors interesados registran/desregistran en un `UWorldSubsystem` en BeginPlay/EndPlay. La query es O(1).
- **`IGameplayTagAssetInterface`**: si el filtro es por tag.

```cpp
// ✓ FIX: registry subsystem
class UEnemyRegistry : public UWorldSubsystem {
    UPROPERTY()
    TArray<TObjectPtr<AEnemy>> Enemies;
public:
    void Register(AEnemy* E) { Enemies.AddUnique(E); }
    void Unregister(AEnemy* E) { Enemies.Remove(E); }
    const TArray<TObjectPtr<AEnemy>>& GetAll() const { return Enemies; }
};
```

### 2. Cast hell en Blueprints (load chains)

```
BP_Player → Cast to BP_GameMode → Cast to BP_UIController → Cast to BP_HUD
                     ↑                       ↑                       ↑
              Cada Cast HARD-LOADEA           todo este chain al BP que castea
```

**Por qué malo**: `Cast<BP_Foo>` en BP es una hard reference. Cargar BP_A que castea a BP_B carga BP_B con todas sus deps. Un BP "casteando" a 5 otros lleva 50 BPs al RAM.

**Fix moderno**:
- **Interfaces (Blueprint Interfaces)**: en lugar de "cast a BP_Damageable y llamar TakeDamage", definí `BPI_Damageable` con `TakeDamage`. Cualquier BP implementa la interface, no hay hard ref.
- **GameplayTags**: en lugar de "cast a BP_Enemy", chequea `HasTag(GameplayTag.Enemy)`.
- **Soft references**: cuando necesitás referir a otro asset sin loadearlo, soft ref + async load.
- **Eventos / dispatchers**: en lugar de Cast→Call, dispatcher en el subject + binding desde el observer.

### 3. Tick on everything

```cpp
// ❌ por default después de migrarte de UE4, todos los actores tickean
AActor::AActor() {
    PrimaryActorTick.bCanEverTick = true;   // off es better default
}
```

**Por qué malo**: 10000 actores tickeando cada frame, 90% no necesitan. Game thread saturated.

**Fix moderno**:
- **OFF por default**. UE5 lo hace por vos. NO lo re-actives sin razón.
- **TimerManager** para schedules: `GetWorldTimerManager().SetTimer(...)` cada N segundos.
- **Events**: si solo reaccionás a algo, usar events en lugar de polling con tick.
- **TickInterval**: si necesitás tick pero no cada frame, `PrimaryActorTick.TickInterval = 0.5f` (cada 500ms).
- **Component-specific tick**: a veces no necesita tickear EL ACTOR ENTERO, solo un component.

### 4. Hardcoded asset paths con `LoadObject` o `FindObject`

```cpp
// ❌ ANTI-PATTERN
UStaticMesh* Mesh = LoadObject<UStaticMesh>(nullptr, TEXT("/Game/Meshes/Cube.Cube"));
```

**Por qué malo**: string path = fragile (rename rompe), no validación de compile-time, no GC-friendly.

**Fix moderno**:
- **`@editable` field**: exponé el asset al editor, designer dragea referencia. UPROPERTY + soft ref si es lazy.
- **PrimaryDataAsset** con AssetManager si es asset gestionado por ID.

```cpp
UPROPERTY(EditDefaultsOnly)
TSoftObjectPtr<UStaticMesh> MyMesh;   // designer asigna, no hardcoded
```

### 5. Monolithic GameMode / Character

```
ABaseCharacter (5000 líneas)
├── Movement logic
├── Combat logic
├── Inventory logic
├── Dialogue logic
├── Save/load logic
└── ...
```

**Por qué malo**: violation de SRP, imposible de testear, BP-too-big-to-load.

**Fix moderno**:
- **Component composition**: cada responsabilidad → UActorComponent.
  - `UCombatComponent`, `UInventoryComponent`, `UDialogueComponent`.
- **Subsystems** para state global que no pertenece a un actor.
- **Data-driven**: stats / config en DataAssets.

### 6. BP-to-BP cast chains (variante de #2)

```
BP_Hud → Cast to BP_PlayerController → Cast to BP_GameState → ...
```

**Fix moderno**: **Subsystem como mediator**. Cada BP que necesita global state lo pregunta al subsystem (que SÍ es global y NO requiere cast chain).

```cpp
UGameInstanceSubsystem* MySys = GetGameInstance()->GetSubsystem<UMyGameSubsystem>();
int Score = MySys->GetScore();
```

### 7. `GetWorld()` en destructores

```cpp
AMyActor::~AMyActor() {
    GetWorld()->...   // ❌ GetWorld() puede ya ser null/garbage en destructor
}
```

**Por qué malo**: destructor corre durante GC, world puede estar siendo destruido también.

**Fix**: usar `EndPlay` para cleanup que necesita world. Destructor solo para liberar memoria propia.

### 8. Variables públicas en Blueprints (cross-BP coupling)

BPs accediendo a variables públicas de otros BPs vía Cast → mismo problema que #2.

**Fix**: getters/setters (functions) en la BP target + interface si es cross-class.

## Detectores rápidos

Patrones a buscar al hacer review:

| Patrón regex / visual | Anti-pattern probable |
|---|---|
| `GetAllActors` | #1 (registry candidate) |
| `Cast<UMyBP_*>` en BP | #2 (cast hell) |
| `bCanEverTick = true` sin justificación | #3 (tick abuse) |
| `LoadObject<...>("/Game/...")` | #4 (hardcoded path) |
| Clase / BP con >1000 líneas o >50 nodes | #5 (monolith) |
| Cadenas de 3+ Cast nodes en BP | #2 + #6 (cast chain) |
| `GetWorld()` en `~Foo()` o `BeginDestroy` | #7 |

## Workflow

1. **Recibí el código o BP graph** (screenshot, código, descripción).
2. **Pasada 1**: scan por los 8 anti-patterns. Marcar cada hallazgo con severity.
3. **Pasada 2**: para cada anti-pattern detectado, propose el fix moderno con código de ejemplo.
4. **Reportá** agrupado, ordenado por impact (perf > coupling > load).

## Output esperado

```
## UE anti-patterns review — `BP_PlayerHUD`

### 🚨 Anti-patterns serios
- **#2 Cast hell**: `BP_PlayerHUD` castea a `BP_PlayerController`, que castea a `BP_GameState`, que castea a `BP_DataManager`. Cargar HUD = cargar 4 BPs + sus deps. Reference viewer muestra ~30 hard refs.
  → Fix: `UMyGameSubsystem::Get(this)->GetCurrentData()`. Subsystem en lugar de cast chain.

- **#1 GetAllActorsOfClass en Tick**: línea 45 hace `GetAllActorsOfClass(AEnemy)` cada tick. Con 200 enemies y 5 huds, eso es 200×5×60 = 60k ops/seg.
  → Fix: registry pattern. Crear `UEnemyRegistry : UWorldSubsystem`, enemies se registran en BeginPlay, HUD consulta O(1).

### ⚠️ Improvements
- **#3 Tick abuse**: HUD tickeando cada frame para actualizar contador de score. Score cambia ~1 vez por minuto.
  → Fix: bind a Dispatcher `OnScoreChanged` desde subsystem, update solo cuando cambia.

### Resumen
2 serios (cast hell + GetAllActors tick), 1 leve. Aplicando los 2 primeros: load time de la UI baja drásticamente + tick load del game thread baja también.
```

## What this skill is NOT for

- Anti-patterns de otro stack (esto es UE-specific).
- Performance tuning sin que sea un anti-pattern (eso es `ue-profiling-and-insights`).
- Linting automático (eso requiere tooling, no review human).
- Implementing los fixes (esta skill identifica + propone; implementación es trabajo aparte).
