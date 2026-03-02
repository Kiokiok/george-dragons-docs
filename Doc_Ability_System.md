# Ability System Documentation

## Overview

The Ability System is a flexible framework for defining combat actions that pawns can perform. Abilities can deal damage, apply effects, move pawns, or trigger any custom logic through the Battle Action system.

## Core Components

### BaseAbility
The abstract base class all abilities inherit from. Abilities define:

- **Casting validity** (`CanPawnCast`) - Can the pawn cast this ability? (checks AP, status effects, cooldowns)
- **Cast area** (`GetCastArea`) - Which cells can the ability be cast from?
- **Effect area** (`GetEffectArea`) - Which cells are affected by the ability?
- **Execution** (`CastAbility`) - What does the ability actually do?

### BattlePawn
The pawn executing the ability. Abilities interact with pawns through:

- **Stats**: `CurrentAP` (action points), `CurrentMovementPoints`, `CurrentHP`
- **State**: Can they move? Do they have status effects preventing actions?
- **Events**: Equipment and effects can modify outgoing damage/effects through the pawn's event system

### BattleInstance
The rules engine. Abilities call high-level action methods instead of directly mutating state:

- `DealDamage(packet)` - Processes damage through event pipeline
- `Heal(packet)` - Heals a target with modifiers
- `ApplyEffect(effect)` - Adds a status effect
- `RemoveEffect(effect)` - Removes a status effect
- `MovePawn(request)` - Moves pawn via pathfinding
- `TeleportPawn(request)` - Teleports pawn (swap-safe)
- `PushPawn(request)` - Pushes pawn (terrain-aware)

### DamagePacket
Intent-based damage representation with modifiable properties:

```
DamagePacket {
  SourcePawn, TargetPawn
  BaseAmount, DamageType (Fire, Blunt, etc.)
  FinalAmount (modified by events)
  Flags: IsDotTick, IsAbilityDamage, CanCrit, IgnoreArmor
}
```


The packet flows through source and target pawn events, allowing modifiers (equipment, status effects) to alter the final damage before it's applied.

### StatusEffect System
Status effects represent buffs, debuffs, and DoTs:

- **DoT Effects** - Damage over time with configurable tick timing (start/end of turn) and stacking rules
- **Movement Debuffs** - Reduce movement points
- **Movement Prevention** - Prevent movement entirely (root/stun)
- **Custom Effects** - Extend `StatusEffect` for custom mechanics

Effects are managed per-pawn via `StatusController` and process automatically at turn start/end.

### Equipment System
Equipment items subscribe to pawn events to modify ability outcomes:

```
Equipped → OnEquipped() → Subscribe to pawn events
Cast Ability → Event fires → Equipment modifies result
Unequipped → OnUnequipped() → Unsubscribe from events
```


Example: Ring of Flames adds +2 to all fire damage by listening to `OnBeforeDealDamage`.

## Ability Authoring Workflow

1. **Extend BaseAbility**
2. **Implement CanPawnCast** - Validate AP, MP, status conditions
3. **Implement GetCastArea** - Define valid targeting range
4. **Implement GetEffectArea** - Define what cells are affected
5. **Implement CastAbility** - Call `BattleInstance` actions:
```csharp
public override void CastAbility(BattlePawn pawn, BattleInstance battle, Vector2I target)
   {
       var targetPawn = battle.GetPawnAtCell(target);
       battle.DealDamage(new DamagePacket(pawn, targetPawn, 10, DamageType.Elemental_Fire));
       pawn.CurrentAP -= 2; // Spend AP
   }
```


## Event Pipeline

Every ability action goes through an event pipeline:

1. **Before Event** - Source/Target pawns and equipment can modify the action
2. **Action Applied** - State is updated
3. **After Event** - Pawns and equipment can react (triggers, logging, VFX)

This allows complex interactions without modifying ability code.

## Key Design Principles

✅ **Abilities don't mutate state directly** - They call BattleInstance action methods  
✅ **Modifiers are decoupled** - Equipment and effects hook into events, not ability code  
✅ **Centralized rules** - All combat logic in BattleInstance ensures consistency  
✅ **Extensible** - New equipment, status effects, and abilities don't require changes to existing code