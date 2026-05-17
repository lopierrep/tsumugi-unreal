---
name: ue-enhanced-input
description: Sistema Enhanced Input de UE5 — reemplazo del legacy InputAction/InputAxis. Conoce los 4 conceptos (InputAction, InputMappingContext, Trigger, Modifier), cómo registrar IMC en PlayerController, el binding por ETriggerEvent (Started, Triggered, Completed, Ongoing, Canceled), prioridades de IMC stacking, modifiers (DeadZone, Negate, Swizzle), triggers (Pressed, Released, Hold, Tap, Chord), y los pitfalls (no setear IMC, binding equivocado de event, modifiers sin sentido).
---

# UE Enhanced Input

Sistema de input moderno en UE5 (4.26+, default desde UE5.1). Reemplaza el legacy `InputAction` / `InputAxis` con un modelo más flexible: contexts apilables, triggers customizables, modifiers, y rebinding runtime sin recompile.

## Los 4 conceptos

| Concepto | Qué es | Asset type |
|---|---|---|
| **InputAction (IA)** | Acción semántica abstracta. NO tiene tecla. (e.g., "IA_Jump", "IA_Move", "IA_Fire") | `UInputAction` |
| **InputMappingContext (IMC)** | Mapeo de teclas/sticks → InputActions. Apilable y prioritizable. | `UInputMappingContext` |
| **Trigger** | Cuándo se dispara la IA: pressed, released, hold, tap, chord, etc. | Settings en IMC |
| **Modifier** | Cómo se transforma el input value: deadzone, scale, negate, swizzle xy↔yx, etc. | Settings en IMC |

**Regla mental**: la IA dice "QUÉ querés que pase" (jump, fire, look), el IMC dice "QUÉ tecla lo dispara". Cambiar IMC = cambiar binding sin tocar la lógica.

## Setup mínimo end-to-end

### 1. Crear IA en editor
- Click derecho en Content Browser → Input → Input Action.
- Settear `ValueType`: `Bool` (toggle/press), `Axis1D` (float), `Axis2D` (Vector2 — stick), `Axis3D` (Vector — raro).

### 2. Crear IMC y agregar mappings
- Click derecho → Input → Input Mapping Context.
- En el IMC, **+ Mappings**, asignar IA, asignar tecla(s).
- Por mapping podés agregar Triggers y Modifiers.

### 3. Registrar IMC en PlayerController (o Pawn)

```cpp
#include "EnhancedInputSubsystems.h"
#include "EnhancedInputComponent.h"

UCLASS()
class AMyPlayerController : public APlayerController
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category="Input")
    TObjectPtr<UInputMappingContext> DefaultMappingContext;

    UPROPERTY(EditDefaultsOnly, Category="Input")
    TObjectPtr<UInputAction> JumpAction;

    UPROPERTY(EditDefaultsOnly, Category="Input")
    TObjectPtr<UInputAction> MoveAction;

protected:
    virtual void BeginPlay() override;
    virtual void SetupInputComponent() override;

    void HandleJump(const FInputActionValue& Value);
    void HandleMove(const FInputActionValue& Value);
};

void AMyPlayerController::BeginPlay()
{
    Super::BeginPlay();

    if (UEnhancedInputLocalPlayerSubsystem* Subsystem =
        ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(GetLocalPlayer()))
    {
        Subsystem->AddMappingContext(DefaultMappingContext, /*Priority=*/0);
    }
}

void AMyPlayerController::SetupInputComponent()
{
    Super::SetupInputComponent();

    if (UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(InputComponent))
    {
        EIC->BindAction(JumpAction, ETriggerEvent::Started, this, &AMyPlayerController::HandleJump);
        EIC->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AMyPlayerController::HandleMove);
    }
}

void AMyPlayerController::HandleMove(const FInputActionValue& Value)
{
    FVector2D MoveVector = Value.Get<FVector2D>();
    // MoveVector.X = horizontal, MoveVector.Y = vertical
    // Usar para mover el pawn
}

void AMyPlayerController::HandleJump(const FInputActionValue& Value)
{
    // Disparado en Started, value no importa para acción boolean
    if (APawn* P = GetPawn())
    {
        if (ACharacter* C = Cast<ACharacter>(P)) C->Jump();
    }
}
```

## ETriggerEvent: cuándo se dispara tu callback

| Event | Cuándo |
|---|---|
| `Started` | El input acaba de empezar (frame 1 de tecla apretada) |
| `Ongoing` | El input sigue pero todavía no satisface el trigger (e.g. Hold 1s, 500ms in) |
| `Triggered` | El trigger se satisfizo (Pressed: cada frame mientras apretado; Hold 1s: cuando se cumple) |
| `Completed` | El input terminó exitosamente (tecla soltada después de satisfacer trigger) |
| `Canceled` | El input se canceló antes de satisfacer trigger (e.g. Tap: soltó demasiado tarde) |

**Patrones típicos**:
- Jump: `BindAction(IA_Jump, ETriggerEvent::Started, ...)` — disparar una vez al apretar.
- Movement: `BindAction(IA_Move, ETriggerEvent::Triggered, ...)` — cada frame mientras input != 0.
- Charge attack: `BindAction(IA_Attack, ETriggerEvent::Completed, ...)` con trigger Hold — disparar al soltar después de cargar.

## Triggers built-in

| Trigger | Comportamiento |
|---|---|
| `Pressed` (default) | Triggered cada frame mientras apretado |
| `Released` | Triggered un frame cuando se suelta |
| `Hold` | Triggered cuando se cumple el tiempo de hold |
| `Hold And Release` | Como Hold, pero el Triggered ocurre al RELEASE después del hold |
| `Tap` | Triggered si se suelta antes de N segundos (quick tap) |
| `Pulse` | Triggered cada N segundos mientras apretado |
| `Chorded Action` | Triggered solo si otra IA está activa simultáneamente (combos) |

## Modifiers built-in

| Modifier | Qué hace |
|---|---|
| `Dead Zone` | Inputs analógicos por debajo de threshold → 0 |
| `Scalar` | Multiplica el value por una constante |
| `Negate` | value × -1 (útil para invertir Y de mouse) |
| `Swizzle` | Reordena los axes (YXZ, XZY, etc.) |
| `To World Space` | Convierte input local del actor a world space |
| `Smooth Delta` | Suaviza el input a lo largo de N frames |

## IMC stacking & priorities

Podés tener múltiples IMCs activos. Higher priority overrides lower:

```cpp
// Default gameplay (priority 0)
Subsystem->AddMappingContext(DefaultMappingContext, 0);

// Cuando entrás a un vehículo, agregás el IMC del vehicle (priority 1)
Subsystem->AddMappingContext(VehicleMappingContext, 1);

// Cuando salís
Subsystem->RemoveMappingContext(VehicleMappingContext);
```

**Pattern típico**:
- Priority 0: gameplay base.
- Priority 1: contextos especiales (vehicle, swimming, climbing).
- Priority 10: UI / menu (bloquea gameplay inputs).

## Pitfalls comunes

### 1. Olvidar agregar el IMC

Síntoma: el callback nunca se dispara. Causa: no llamaste `AddMappingContext`. Verificá BeginPlay del PlayerController.

### 2. Bindear al ETriggerEvent equivocado

- Movement bindeado a `Started` → solo se mueve un frame al apretar la tecla, no continuo. Fix: `Triggered`.
- Jump bindeado a `Triggered` → spam de jump cada frame mientras apretado. Fix: `Started`.

### 3. ValueType incorrecto en la IA

IA con `ValueType=Bool` pero callback hace `Value.Get<FVector2D>()` → siempre 0. Match al ValueType correcto.

### 4. Modifiers en orden equivocado

Modifiers se aplican en orden. `Negate → Scalar 2` ≠ `Scalar 2 → Negate`. Si el resultado es raro, revisá el orden.

### 5. Mismo IMC en PlayerController y Pawn

Si registrás IMC en ambos, los triggers se duplican. Elegí un solo lugar (PlayerController para player input típicamente, Pawn solo si querés que se desactive automáticamente al despossesar).

### 6. Asumir que `LookAxis` y `Move` son la misma escala

Mouse vs stick analógico vs teclado tienen escalas distintas. Modifiers `Scalar` y `Smooth Delta` ayudan a normalizar. No hardcodear sensibilidades sin tener en cuenta el dispositivo.

## Migración desde legacy Input

| Legacy | Enhanced Input |
|---|---|
| `InputAction "Jump"` en DefaultInput.ini | `UInputAction` + entry en `UInputMappingContext` |
| `InputAxis "MoveForward"` | `UInputAction` con `ValueType=Axis1D` o `Axis2D` |
| `InputComponent->BindAction("Jump", IE_Pressed, ...)` | `UEnhancedInputComponent->BindAction(IA_Jump, ETriggerEvent::Started, ...)` |
| `InputComponent->BindAxis("MoveForward", ...)` | `BindAction(IA_Move, ETriggerEvent::Triggered, ...)` |

Migración recomendada por iteraciones — UE soporta ambos simultáneamente.

## What this skill is NOT for

- Rebinding runtime (UI de "configurar controles") — eso es trabajo encima, no parte del sistema base.
- Input recording / replay.
- Force feedback / haptics (otra API).
- Touch / VR controllers (extensiones, no covered acá).
