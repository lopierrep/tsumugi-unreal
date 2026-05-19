---
name: ue-actor-lifecycle
description: Use whenever the user is deciding which init hook to use, debugging "X is null in Constructor", confused about Constructor vs BeginPlay vs OnConstruction, or working with init order for components/actors. Triggers on phrases like "en qué función inicializo esto", "constructor vs BeginPlay", "el componente no está listo en X", "OnConstruction vs PostInitializeComponents", "ciclo de vida del actor", "actor lifecycle order", "where do I initialize X", "BeginPlay vs constructor", "PostInitializeComponents timing", "component not ready in constructor". Returns the canonical lifecycle order + what's safe in each hook + PIE/editor/cooked differences. UE5-specific.
---

# UE Actor Lifecycle

Subtle initialization bugs son la #2 source de UE bugs. "¿Por qué `GetWorld()` crashea en el constructor?" → wrong init function. Cada gameplay programmer redescubre esto dolorosamente. Esta skill es la referencia.

## El orden canónico

Para un actor instanciado en un level cargado:

```
1. Constructor                          (en CDO + en cada instance)
2. PostInitProperties                   (después de constructor)
3. OnConstruction                       (cuando el actor aparece en el world, también re-corre en editor)
4. PreInitializeComponents              (justo antes de los components)
5. InitializeComponent                  (uno por cada UActorComponent)
6. PostInitializeComponents             (después de TODOS los components inicializados)
7. BeginPlay                            (cuando empieza el juego/PIE)
8. Tick                                 (cada frame, si bCanEverTick)
9. EndPlay                              (cuando el actor sale del play)
10. BeginDestroy                        (lifecycle de UObject, async)
11. FinishDestroy                       (después de BeginDestroy completa)
```

## Qué es SAFE en cada hook

### Constructor

**Propósito**: setup de defaults, creación de subcomponents declarados como UPROPERTY, configuración de objects que NO requieren world.

**Safe**:
- `CreateDefaultSubobject<UMeshComponent>("Mesh")` para crear sub-components.
- Setear defaults: `bReplicates = true`, `PrimaryActorTick.bCanEverTick = false`.
- Asignar UPROPERTY defaults: `MaxHealth = 100.f`.

**UNSAFE — NO HACER**:
- `GetWorld()` — el actor no está en un world todavía. Puede devolver `nullptr` o un editor world transitorio.
- `GetGameInstance()` — same problem.
- Subscribirse a eventos (no hay world).
- Loadear assets sync.
- Spawn de otros actores.

**Gotcha del CDO** (Class Default Object): el constructor corre PRIMERO para el CDO (objeto template que vive en memoria desde startup), DESPUÉS para cada instance. En el CDO, NO tenés world ni outer útil. Tu código debe ser tolerante a ambos casos.

### PostInitProperties

**Cuándo**: justo después del constructor.

**Útil para**: setup que depende de UPROPERTY values siendo finalizados (después de defaults Y de cualquier override por el editor).

**Raramente necesitás overridearlo** salvo casos avanzados.

### OnConstruction

**Cuándo**: cuando el actor "aparece" en el world. En el editor, corre EACH TIME que tocás una property del actor en el detalles panel (re-construction). En runtime, corre antes de PreInitializeComponents.

**Safe**:
- `GetWorld()` SÍ funciona.
- Generar geometría procedural basada en UPROPERTY values.
- Setup que el usuario quiera ver actualizado en el editor sin play.

**UNSAFE**:
- Spawn de otros actores (en editor te llena el level de actores).
- Subscribirse a eventos persistentes (se re-suscribe cada edit).
- Side effects no idempotentes.

**Pitfall**: usuarios olvidan que OnConstruction corre en editor y meten side effects que ensuician el level.

### PreInitializeComponents

**Útil**: setup que debe pasar ANTES de que los components se inicialicen. Raramente necesitás overridearlo.

### InitializeComponent (en UActorComponent)

**Cuándo**: para cada component, justo después de creado. **Pero solo se llama si el component tiene `bWantsInitializeComponent = true`** (no default en UE 5+; verificá).

**Útil**: setup del component-specific en sí, no del owner.

### PostInitializeComponents

**Cuándo**: después de que TODOS los components del actor están inicializados.

**Safe**:
- Acceder a sub-components: `GetMesh()`, `GetCharacterMovement()` están listos.
- Setup que coordina entre components.

### BeginPlay

**Cuándo**: when the game actually starts for this actor. En multiplayer, en el lado server primero, después se replica a clientes.

**Safe**:
- TODO el world está cargado.
- Subscribirse a eventos.
- Spawn de otros actors.
- Empezar timers.
- Lookup de otros actors.

**Pitfall #1 — orden entre actors NO determinístico**: BeginPlay de actor A puede correr antes o después de BeginPlay de actor B. NO confíes en orden.

**Fix**: si necesitás orden estricto, usá un coordinador (GameMode subsystem, Subsystem global). El coordinator hace su BeginPlay y notifica a los demás cuando todos están listos.

**Pitfall #2 — ListenServer host vs cliente**: en listen server, el "host" es tanto server como cliente. BeginPlay corre 1 vez. En cliente puro, corre 1 vez también. En dedicated, server-side único.

### Tick

**Cuándo**: cada frame, si `PrimaryActorTick.bCanEverTick = true`.

**Default OFF para nuevos actors a partir de UE 5.0** (perf-conscious default). Si necesitás tick, settealo en constructor:
```cpp
PrimaryActorTick.bCanEverTick = true;
PrimaryActorTick.TickInterval = 0.1f;  // cada 100ms en lugar de cada frame
```

**Anti-pattern**: tick en TODO. Usá `FTimerManager` para callbacks scheduled (cada N segundos), events para reactive, tick solo si literalmente necesitás cada frame (movement, animation poses, sensor sampling).

### EndPlay

**Cuándo**: cuando el actor sale del play. Recibís un `EEndPlayReason`:

| Reason | Cuándo |
|---|---|
| `Destroyed` | Llamaste `Destroy()` o GC lo agarró |
| `LevelTransition` | OpenLevel a otro nivel |
| `EndPlayInEditor` | Stopped PIE |
| `RemovedFromWorld` | Streaming unloaded el sublevel |
| `Quit` | App is closing |

**Útil para**: limpieza, unsubscribe de eventos, save state.

**No safe**: spawn de actores (estás muriendo).

### BeginDestroy / FinishDestroy

Lifecycle de UObject base (no de Actor). Casi nunca lo overriedeás — solo para UObjects custom con cleanup pesado.

## Cheatsheet visual

```
+----------------------+----------------------+----------------------+
| Hook                 | Safe                 | NOT Safe             |
+----------------------+----------------------+----------------------+
| Constructor          | UPROPERTY defaults,  | GetWorld(),          |
|                      | CreateDefaultSubobject| events, spawn       |
+----------------------+----------------------+----------------------+
| OnConstruction       | GetWorld() ok,       | spawn actors (corre  |
|                      | procedural geometry  | en editor)           |
+----------------------+----------------------+----------------------+
| PostInitializeComps  | acceder components   | -                    |
+----------------------+----------------------+----------------------+
| BeginPlay            | TODO                 | -                    |
|                      | (subscribe, spawn,   |                      |
|                      |  timers, lookups)    |                      |
+----------------------+----------------------+----------------------+
| Tick                 | per-frame work       | I/O sync, expensive  |
|                      |                      | allocs                |
+----------------------+----------------------+----------------------+
| EndPlay              | cleanup, unsubscribe | spawn actors         |
+----------------------+----------------------+----------------------+
```

## Common errors y fixes

### "GetWorld() returned nullptr in Constructor"

Causa: Constructor corre en CDO + sin world. Solución: mover el código a OnConstruction o BeginPlay.

### "Component is null in BeginPlay"

Posible causa: el component se creó con `CreateDefaultSubobject` (correcto) pero no se asignó al field. Verificá:
```cpp
MyComponent = CreateDefaultSubobject<UMyComponent>("CompName");
```

### "BeginPlay of other actor not run yet when I need it"

Causa: orden de BeginPlay entre actors no determinístico. Solución: usar un Subsystem coordinator o pattern de "ready check" con polling/events.

### "OnConstruction creates duplicate actors in editor"

Causa: spawn de actors en OnConstruction → corre cada vez que tocás una property en editor → spawnea N veces.

Fix: NO spawnees en OnConstruction. Mover a BeginPlay (solo runtime) o agregar guard `if (HasAnyFlags(RF_ClassDefaultObject)) return;`.

## What this skill is NOT for

- Component lifecycle individual (component tiene sus propios hooks; esta skill lo toca tangencialmente).
- Networking lifecycle (eso lo cubre `ue-replication-roles`).
- Subsystem lifecycle (eso es `ue-subsystems`).
- World partition streaming events (separado).
