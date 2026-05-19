---
name: ue-gc-and-pointers
description: Use whenever the user is choosing a reference type in UE5 — TObjectPtr, TWeakObjectPtr, TSoftObjectPtr, raw T*, or non-UObject smart pointers (TSharedPtr/TWeakPtr/TUniquePtr) — or debugging GC-related crashes (dangling pointers, unexpected garbage collection, references not nulling out). Triggers on phrases like "qué tipo de puntero uso para X", "TObjectPtr vs raw pointer", "TWeakObjectPtr cuándo", "esto se está garbage-colectando", "dangling pointer en UE", "memoria UObject", "should this be UPROPERTY", "smart pointers in Unreal", "AddToRoot", "GC crash". Returns a decision matrix for UE5 reference types + GC behavior + common pitfalls. UE5.0+ specifically (TObjectPtr era, no auto-null).
---

# UE GC & Pointers

#1 source de crashes en mid-senior UE devs. Las semánticas de GC en UE5 cambiaron en 5.0 (no auto-null), `TObjectPtr` fue introducido como reemplazo de raw pointers en headers, y la mayoría de Stack Overflow / forums es desactualizado. Esta skill es la decisión matrix correcta para 5.0+.

## Decision matrix

| Tipo | Cuándo usar | GC behavior |
|---|---|---|
| `UPROPERTY() TObjectPtr<T>` | Hard ref a un UObject que querés mantener vivo. **Default para fields en headers.** | Marca el target como reachable (no se garbage-colecta mientras vos lo refies) |
| `UPROPERTY() TWeakObjectPtr<T>` | Ref a un UObject que NO querés mantener vivo (e.g., target del enemigo, cache que puede expirar) | NO mantiene reachable; chequeás `.IsValid()` antes de usar |
| `TSoftObjectPtr<T>` | Ref a un asset que NO querés cargado en memoria automáticamente | Tema de async loading — ver `ue-async-loading` |
| Raw `T*` | Local variables dentro de un método, NUNCA como field de clase | Sin tracking de GC; si el target muere mientras lo usás → dangling |
| `TSharedPtr<T>` / `TWeakPtr<T>` / `TUniquePtr<T>` | Tipos C++ que **NO** son UObject (e.g., `FSlateBrush`, structs pure C++, third-party classes) | RAII clásico — NO interactúan con GC |
| `std::shared_ptr` / `std::unique_ptr` | NO usar en código UE típico | NO play nice con UE GC; usar las TXxxPtr versions |

## La regla maestra: UPROPERTY + TObjectPtr

```cpp
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    // ✓ CORRECTO: UPROPERTY + TObjectPtr en header
    UPROPERTY()
    TObjectPtr<AOtherActor> MyOther;

    // ✗ WRONG: raw pointer field, no UPROPERTY
    AOtherActor* MyOther2;   // GC no lo ve, dangling garantizado al rato

    // ✗ WRONG: UPROPERTY pero raw pointer (UE5+ warning)
    UPROPERTY()
    AOtherActor* MyOther3;   // deprecado en 5.0+, usar TObjectPtr
};
```

### Por qué `UPROPERTY` es crítico

UE GC trackea memoria via **reflection**. El sistema escanea `UPROPERTY()` fields para construir el grafo de "qué refiere a qué". Sin UPROPERTY:
- GC no sabe que tu ref existe.
- Puede colectar el target mientras vos lo refies.
- → dangling pointer → crash.

**Excepción**: variables locales en métodos (stack-allocated raw pointers). Estas no necesitan UPROPERTY porque tu propio método los keeps alive durante su scope.

### TObjectPtr<T> vs raw T* (UE5.0+)

`TObjectPtr` es básicamente `T*` con:
- Better debugging info en el editor.
- Track de "soft references" en cooked builds (futuro).
- Compatible con raw access (`MyPtr->Something()`, `MyPtr.Get()`).
- En `.cpp` files podés usar raw `T*` para variables locales — el `TObjectPtr` es para FIELDS en `.h`.

**Migration rule** (Epic recomienda):
- En headers: `TObjectPtr<T>` para fields UPROPERTY.
- En `.cpp`: raw `T*` para locales (mismo behavior, menos verboso).

## UE5 GC: cuándo corre y qué hace

GC corre periódicamente (default cada 60s, ajustable en `BaseEngine.ini`). En cada pass:

1. **Mark phase**: scan UPROPERTY fields desde root set (objetos marcados como GC-root). Marca todo lo reachable.
2. **Sweep phase**: cualquier UObject NO marcado → `BeginDestroy()` + `IsValidLowLevel() = false` (marked as garbage), eventualmente memoria liberada.

**Cambio importante en 5.0+**: cuando un UObject es garbage-colectado, las refs hacia él NO se setean a nullptr automáticamente. La ref sigue apuntando a la memoria liberada (`MyPtr != nullptr` puede ser true pero `IsValid(MyPtr)` es false).

```cpp
if (MyPtr != nullptr)             // ❌ no es suficiente en 5.0+
    MyPtr->DoSomething();

if (IsValid(MyPtr))               // ✓ chequea ambos: not null AND not garbage
    MyPtr->DoSomething();
```

**Helper**: `IsValid(UObject*)` = `Ptr != nullptr && !Ptr->IsPendingKill() && !Ptr->IsUnreachable()`.

## TWeakObjectPtr: cuándo usarlo

```cpp
// Caso típico: cache que puede expirar
UCLASS()
class AAIController : public AController
{
    UPROPERTY()
    TWeakObjectPtr<APawn> CurrentTarget;   // si el target muere, no lo mantengamos vivo

public:
    void Update() {
        if (APawn* Target = CurrentTarget.Get())   // null si murió
            FireAt(Target);
    }
};
```

**Cuándo SÍ**:
- Refs a actores que vos no ownás (enemy targeting, last damaged actor).
- Refs cíclicas que de otra forma harían leak (parent ↔ child).
- Cache de results que puede invalidarse.

**Cuándo NO**:
- Composición de actor + sus components → composición es ownership, usar `TObjectPtr`.
- Refs a singletons / GameInstance Subsystems → siempre vivos, usar hard ref.

## Non-UObject smart pointers

Para tipos que NO son UObject (FStruct con destructor, third-party classes, Slate widgets):

| Tipo | Caso |
|---|---|
| `TSharedPtr<T>` | Shared ownership; el último que lo suelta destruye. Default para Slate widgets, FSlateBrush, etc. |
| `TWeakPtr<T>` | Observador sin ownership de un TSharedPtr; chequeá `.Pin()` para usar |
| `TUniquePtr<T>` | Sole owner; auto-delete cuando sale de scope. Móvil-only, no copiable |
| `TSharedRef<T>` | Como TSharedPtr pero NUNCA null (referenced-required) |

**NO uses `std::*_ptr` en código UE**: la threading model y el allocator son distintos. Usá las T* versions.

## Pitfalls comunes

### 1. Forgot UPROPERTY → dangling

```cpp
class AMyActor : public AActor {
    AOtherActor* MyOther;   // NO UPROPERTY
};
// MyOther->Foo() → puede crash en cualquier momento si GC corrió
```

**Fix**: agregá `UPROPERTY()` + `TObjectPtr<>`.

### 2. Mixing UE smart pointers y std

```cpp
std::shared_ptr<UMyObject> Ptr;   // ❌ UMyObject es UObject, no podés
```

UObjects se manejan por GC, no por shared_ptr.

### 3. AddToRoot sin balance

```cpp
MyObj->AddToRoot();   // marca como GC-root, never collected
// Después olvidás llamar RemoveFromRoot() → leak permanente
```

**Fix**: solo usar `AddToRoot` cuando realmente necesitás. Para mantener vivo dentro de la lifetime de tu actor, una `TObjectPtr<UMyObject> UPROPERTY` es suficiente.

### 4. TObjectPtr en hot path

`TObjectPtr` tiene una micro-indirection. En hot loops (>1000 calls/frame), `MyTOP.Get()` se vuelve más lento que raw access. Si profilás y aparece, usar raw `T*` local desde el TObjectPtr al entrar al loop:

```cpp
AOtherActor* Cached = MyTopField.Get();
for (int i = 0; i < 10000; i++) {
    Cached->Foo();   // raw access, fast
}
```

### 5. Holding TObjectPtr<APlayerController> across level transitions

Los actors mueren al cambiar nivel. Si guardás un TObjectPtr en un singleton (GameInstance) a un actor, después de OpenLevel ese ptr es garbage. Usar TWeakObjectPtr o re-resolver post-transition.

## Workflow

1. **Identificá la situación**: nuevo field, refactor de ref existente, debugging dangling.
2. **Decision flow**:
   - ¿UObject? → UPROPERTY + TObjectPtr o TWeakObjectPtr (según ownership).
   - ¿Non-UObject? → TSharedPtr / TUniquePtr según ownership.
   - ¿Local en método? → raw `T*` está bien.
   - ¿Asset opcional / pesado / lazy? → TSoftObjectPtr (handoff a `ue-async-loading`).
3. **Validá**: ¿está UPROPERTY-marcado? ¿usás `IsValid()` antes de derefef? ¿chequeás `.Get()` en weak?
4. **Reportá** issues y fix concreto.

## What this skill is NOT for

- Async loading / TSoftObjectPtr (eso es `ue-async-loading`).
- Generational GC tuning (raramente necesario).
- Custom allocator strategies.
- Lifetime de actors / components (eso es `ue-actor-lifecycle`).
