---
name: ue-replication-roles
description: Patrón de replicación en UE5 — los 4 NetRole (Authority, AutonomousProxy, SimulatedProxy, None), los 3 NetMode (Standalone, ListenServer, DedicatedServer), RPCs (Server/Client/NetMulticast con Reliable/Unreliable), Replicated properties con OnRep_X, conditions (DOREPLIFETIME_CONDITION_NOTIFY), pitfalls (OnRep no se dispara en ListenServer host, HasAuthority check obligatorio antes de mutar state, sync-load en RPCs).
---

# UE Replication Roles & RPCs

Patrón fundamental para multiplayer en UE5: quién es la autoridad, cómo se replica el state, cómo se llaman funciones cross-machine, y los pitfalls que rompen cualquier prototipo.

## Hard facts: roles y modes

### NetMode (la máquina actual)

| Mode | Qué es | Cuándo |
|---|---|---|
| `NM_Standalone` | Sin networking | PIE single-player, single-player game |
| `NM_ListenServer` | Server que también es uno de los clientes | Listen server hosting (host + N clientes remotos) |
| `NM_DedicatedServer` | Solo server, no renderiza | Producción multiplayer típico |
| `NM_Client` | Cliente conectado a un server | Cualquier cliente que NO sea el host |

Chequeo: `GetNetMode()`.

### NetRole (este actor en esta máquina)

| Role | Qué significa | En qué máquina |
|---|---|---|
| `ROLE_Authority` | Esta máquina es la fuente de verdad para este actor | Server siempre. Cliente solo para actores locales (e.g. su input prediction). |
| `ROLE_AutonomousProxy` | Esta máquina controla input pero NO es autoridad | El cliente que posee al pawn |
| `ROLE_SimulatedProxy` | Esta máquina recibe updates pasivamente | Clientes que NO poseen al pawn (replicación pasiva) |
| `ROLE_None` | Actor no replica | Default si `bReplicates=false` |

Chequeo: `GetLocalRole()` y `GetRemoteRole()`. Helper: `HasAuthority()` = `GetLocalRole() == ROLE_Authority`.

**Regla mental**: si vas a mutar state del actor, **siempre** verificá `HasAuthority()` primero — si no es autoridad, la mutación se va a sobrescribir cuando llegue la próxima replicación del server.

## Replicated Properties

Marcar property para que replique automáticamente del server a los clientes:

```cpp
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    AMyActor()
    {
        bReplicates = true;  // toggle global de replicación
    }

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    UPROPERTY(ReplicatedUsing=OnRep_Health)
    float Health = 100.f;

    UPROPERTY(Replicated)
    int32 Score = 0;

    UFUNCTION()
    void OnRep_Health();
};

void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyActor, Health);
    DOREPLIFETIME(AMyActor, Score);
}

void AMyActor::OnRep_Health()
{
    // Disparado en clientes (NO autoridad) cuando Health cambia.
    // El valor viejo está en el parámetro opcional `float OldValue`.
    UE_LOG(LogTemp, Log, TEXT("Health changed to %f"), Health);
}
```

### Conditional replication

Para optimizar: replicar solo a ciertos clientes.

```cpp
DOREPLIFETIME_CONDITION(AMyActor, AmmoInClip, COND_OwnerOnly);          // solo al dueño
DOREPLIFETIME_CONDITION(AMyActor, ScorePosition, COND_SkipOwner);       // a todos menos al dueño
DOREPLIFETIME_CONDITION(AMyActor, ServerOnlyState, COND_InitialOnly);   // solo en spawn inicial
```

Conditions útiles: `COND_OwnerOnly`, `COND_SkipOwner`, `COND_InitialOnly`, `COND_ReplayOnly`, `COND_SimulatedOnly`.

## RPCs (Remote Procedure Calls)

| Specifier | Dirección | Quién llama | Quién ejecuta |
|---|---|---|---|
| `Server` | client → server | Cliente que posee el actor | Server |
| `Client` | server → client | Server | Cliente dueño del actor (no otros) |
| `NetMulticast` | server → todos | Server | Server + todos los clientes con el actor relevant |

Ejemplo:

```cpp
UFUNCTION(Server, Reliable)
void ServerFire(const FVector& Direction);

UFUNCTION(Client, Reliable)
void ClientShowDamage(float Amount);

UFUNCTION(NetMulticast, Unreliable)
void MulticastPlayHitVFX(const FVector& Location);
```

Reliable vs Unreliable:
- **Reliable**: garantiza llegada y orden. Más caro. Usar para state-changing (damage, score, abilities).
- **Unreliable**: best-effort, puede perder paquetes. Más barato. Usar para cosmético (VFX, sonido, animation hint).

**Regla**: no spamees Reliable. Si llenás el reliable buffer, el cliente se desconecta con `BunchOverflow`.

### Validación en Server RPCs

Server RPCs DEBEN tener un validate twin (UE compila error si no):

```cpp
UFUNCTION(Server, Reliable, WithValidation)
void ServerFire(const FVector& Direction);

bool AMyPawn::ServerFire_Validate(const FVector& Direction)
{
    // Sanity check, anti-cheat liviano
    return Direction.IsNormalized();
}

void AMyPawn::ServerFire_Implementation(const FVector& Direction)
{
    // lógica real, autoritativa
}
```

Si validate retorna `false`, el cliente se desconecta. **No es para validación de lógica de game** — es solo para detectar inputs corruptos / cheats triviales.

## Pitfalls comunes

### 1. OnRep no se dispara en ListenServer host

El host (que es server + cliente) cambia la property → server no se "auto-rep" a sí mismo. Por defecto `OnRep_X` no corre en el host.

**Fix**: llamar manualmente la función desde el setter del server, o usar `REPNOTIFY_Always`:

```cpp
DOREPLIFETIME_CONDITION_NOTIFY(AMyActor, Health, COND_None, REPNOTIFY_Always);
// Ahora OnRep_Health se llama tanto en clientes como en el host server al cambiar Health.
```

### 2. Mutar state sin verificar HasAuthority

```cpp
void AMyActor::TakeDamage(float Amount)
{
    Health -= Amount;  // BUG: en cliente, se sobrescribe en el próximo rep tick
}
```

**Fix**:

```cpp
void AMyActor::TakeDamage(float Amount)
{
    if (!HasAuthority()) return;
    Health -= Amount;  // server propaga via OnRep_Health
}
```

O hacer la lógica un Server RPC:

```cpp
void AMyActor::RequestTakeDamage(float Amount)
{
    if (HasAuthority()) ApplyDamageInternal(Amount);
    else ServerTakeDamage(Amount);
}
```

### 3. Sync-loading dentro de un RPC

Server RPC que hace `LoadObject<UTexture2D>(...)` bloquea el server tick → todos los clientes laguean. **Pre-cargar** assets antes (con FStreamableManager async load) o usar Soft references resueltos al spawn.

### 4. Confiar en client state

Cliente reporta "estoy en posición X, recibí damage de Y" → server confía → cheat. Server debe re-validar todo lo que matters (collision, range, line-of-sight, cooldowns).

### 5. NetMulticast en actor no replicado

Si `bReplicates=false`, NetMulticast no llega a los clientes. Validar `bReplicates=true` antes de confiar en multicasts.

### 6. Actor no relevante para el cliente

UE optimiza por relevance (distancia, network LOD, dormancy). Un cliente lejano NO recibe replicación de un actor irrelevante → no le llega la rep ni los multicasts. `bAlwaysRelevant=true` lo fuerza pero es caro.

## What this skill is NOT for

- Replicar subobjetos / components no-Actor (eso requiere overrides de `ReplicateSubobjects`).
- Replay system (es separado).
- Bandwidth profiling (`Stat Net`).
- Push Model replication (UE5 feature opcional, futuro skill).

## Cuándo NO usar replicación nativa

- State puramente cosmético en cada cliente (e.g., partícula local).
- Singleplayer puro.
- State que cambia 60 veces por segundo y NO requiere consistencia (tirá un movement prediction custom en lugar de DOREPLIFETIME).
