---
name: ue-gameplay-tags
description: Sistema de Gameplay Tags en UE5 — tags jerárquicos strongly-typed para state, damage types, abilities, ownership, AI states, etc. Reemplazo de strings/enums hard-coded. Conoce la diferencia entre Tag vs TagContainer, las operaciones (HasTag, HasAllTags, HasAnyTags, MatchesTag), el matching jerárquico ("Damage.Fire" matchea "Damage"), config en DefaultGameplayTags.ini, y cuándo NO usarlos (cuando hace falta valor numérico, cuando son <5 alternativas estáticas).
---

# UE Gameplay Tags

Sistema built-in de UE para tags jerárquicos strongly-typed. Reemplaza strings hard-coded (`"player_state_dead"`), enums dispersos, y patrones de "if statement chains" con un sistema central, validado y editable desde editor.

## Por qué usarlos (vs strings o enums)

| Approach | Problemas |
|---|---|
| Strings hard-coded (`if (state == "dead")`) | Typos no caught, no autocomplete, refactor a mano, no jerarquía |
| Enums (`enum EDamageType { Fire, Cold, ... }`) | Listado en código, agregar tag requiere recompile + restart, no jerarquía, no extensible |
| `FName` direct | Igual que enums + menos seguro |
| **Gameplay Tags** | Definidos en `.ini` (sin recompile), autocomplete en editor, jerarquía nativa, validation, exportables para data-driven design |

## Conceptos clave

| Concepto | Qué es |
|---|---|
| `FGameplayTag` | Un tag individual (e.g., `Damage.Fire`) |
| `FGameplayTagContainer` | Conjunto de tags. Un Actor / Component / Ability puede tener muchos |
| `FGameplayTagQuery` | Query estructurada ("hasAny[A,B] AND hasNone[C]") para gating complejo |

## Jerarquía y matching

Los tags son jerárquicos por convención de naming con puntos:
- `Damage`
- `Damage.Fire`
- `Damage.Fire.Ignite`
- `Damage.Cold`

Operaciones de matching:

| Operación | Comportamiento |
|---|---|
| `Tag.MatchesTag(Other)` | exact o descendiente. `Damage.Fire.MatchesTag(Damage)` = **true** |
| `Tag.MatchesTagExact(Other)` | solo exact. `Damage.Fire.MatchesTagExact(Damage)` = **false** |
| `Container.HasTag(Tag)` | container contiene Tag o un descendiente |
| `Container.HasTagExact(Tag)` | container contiene Tag exact |
| `Container.HasAll(Other)` | container contiene TODOS los de Other (con jerarquía) |
| `Container.HasAny(Other)` | container contiene AL MENOS UNO de Other |
| `Container.HasAllExact(Other)` / `HasAnyExact(Other)` | versiones exact |

**Regla mental**: querés que "Damage.Fire" cuente como "Damage" para tu lógica → usá la versión NO-exact. Querés disambiguar entre parent y children → usá exact.

## Cómo declarar tags

**Opción A — desde Project Settings (recomendado)**:
1. Editor → Project Settings → GameplayTags → "Manage Gameplay Tags"
2. Agregar via UI. Se guardan en `Config/DefaultGameplayTags.ini`.

**Opción B — desde código (C++)**:
```cpp
// MyGameplayTags.h
#pragma once
#include "NativeGameplayTags.h"

namespace MyGameplayTags
{
    UE_DECLARE_GAMEPLAY_TAG_EXTERN(Damage_Fire);
    UE_DECLARE_GAMEPLAY_TAG_EXTERN(Damage_Cold);
    UE_DECLARE_GAMEPLAY_TAG_EXTERN(State_Dead);
}

// MyGameplayTags.cpp
namespace MyGameplayTags
{
    UE_DEFINE_GAMEPLAY_TAG_COMMENT(Damage_Fire, "Damage.Fire", "Damage de fuego");
    UE_DEFINE_GAMEPLAY_TAG_COMMENT(Damage_Cold, "Damage.Cold", "Damage de frío");
    UE_DEFINE_GAMEPLAY_TAG_COMMENT(State_Dead, "State.Dead", "Personaje muerto");
}
```

**Opción C — en `.ini` directo**:
```ini
[/Script/GameplayTags.GameplayTagsSettings]
+GameplayTagList=(Tag="Damage.Fire",DevComment="Damage de fuego")
+GameplayTagList=(Tag="Damage.Cold",DevComment="Damage de frío")
```

## Uso en código

**Setear tag en un Actor**:
```cpp
UCLASS()
class AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Tags")
    FGameplayTagContainer Tags;
};

// runtime
FGameplayTag DeadTag = FGameplayTag::RequestGameplayTag(FName("State.Dead"));
MyCharacter->Tags.AddTag(DeadTag);

if (MyCharacter->Tags.HasTag(MyGameplayTags::State_Dead))
{
    // ...
}
```

**Query estructurada** (para condiciones complejas):
```cpp
FGameplayTagQuery Query = FGameplayTagQuery::BuildQuery(
    FGameplayTagQueryExpression()
        .AllTagsMatch()
        .AddTag(MyGameplayTags::State_Dead)
        .AddTag(MyGameplayTags::Damage_Fire)
);

if (Query.Matches(MyCharacter->Tags))
{
    // murió de fuego
}
```

## Casos de uso típicos

| Caso | Tags ejemplo |
|---|---|
| Damage types | `Damage.Fire`, `Damage.Cold`, `Damage.Physical.Blunt` |
| Character state | `State.Dead`, `State.Stunned`, `State.Invisible` |
| Abilities (GAS) | `Ability.Movement.Dash`, `Ability.Combat.Slash` |
| Loot rarity | `Rarity.Common`, `Rarity.Rare`, `Rarity.Legendary` |
| Faction / team | `Faction.Player`, `Faction.Enemy.Goblin` |
| Input context | `Input.Context.Menu`, `Input.Context.Combat` |
| AI behavior | `AI.State.Patrol`, `AI.State.Combat.Aggressive` |

## Anti-patrón: cuando NO usar Gameplay Tags

- **Necesitás valor numérico** (`damage = 50`). Tags son booleanos. Un Actor tiene un tag o no — no tiene "tag con valor 50". Para valores: `UCurveTable`, `UDataTable`, o struct con tag + float.
- **<5 alternativas fijas que no van a crecer**: enum tradicional sigue siendo OK (más liviano).
- **Estado que cambia 60 veces por segundo**: tags son OK pero hay overhead. Para hot-path AI considerá `uint8` bitflags.

## Integración con otros sistemas

- **Gameplay Ability System (GAS)**: GAS está construido sobre Gameplay Tags. Si vas a usar GAS, tags ya están.
- **AI Perception**: filtrar por tags ("ver solo enemigos con `Faction.Enemy`").
- **Damage system**: tag damage events con `Damage.<tipo>` para resistencias / inmunidades.
- **Animation**: notify states pueden setear/limpiar tags durante el play (`State.Animation.Attacking`).

## Pitfalls comunes

- **Re-request del tag en hot loop**: `FGameplayTag::RequestGameplayTag(FName("..."))` hace lookup por nombre. Cachealo en static var o usá `UE_DEFINE_GAMEPLAY_TAG_COMMENT` (resuelve en startup).
- **Jerarquía mal pensada**: empezar con tags planos (`DamageFire`) y después tener que renombrar a `Damage.Fire`. Pensá jerarquía desde el inicio.
- **Mezclar tag con tag exact en lógica de inmunidad**: típica fuente de bugs (Fire damage no es bloqueado por `Damage` inmunidad porque usaste `HasTagExact` por error).
- **No exportar tags al INI**: si los declarás solo en código C++, no podés agregar tags desde el editor sin recompile.

## What this skill is NOT for

- Implementar un damage system completo → otra skill (`ue-damage-system-pattern`).
- Setup completo de Gameplay Ability System → otra skill (`ue-gas-introduction`).
- Decidir si usás enums vs tags para un caso particular — solo enseña el patrón.
