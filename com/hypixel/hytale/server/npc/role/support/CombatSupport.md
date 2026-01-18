---
description: Architectural reference for CombatSupport
---

# CombatSupport

**Package:** com.hypixel.hytale.server.npc.role.support
**Type:** Transient State Object

## Definition
```java
// Signature
public class CombatSupport {
```

## Architecture & Concepts

The CombatSupport class is a stateful component that encapsulates the combat-related state and logic for a parent NPCEntity. It is not a standalone system but rather a dedicated helper object, tightly coupled to the lifecycle of a single NPC.

Its primary architectural purpose is to decouple the concerns of combat execution from the higher-level AI and behavior logic within an NPC's Role. It achieves this by managing three core responsibilities:

1.  **Attack State Machine:** It tracks the active attack sequence (via InteractionChain) and enforces cooldowns (attackPause), preventing the AI from initiating new attacks while one is already in progress.
2.  **Damage Rule Evaluation:** It serves as a queryable rule engine for the server's damage system. The getCanCauseDamage method determines if the parent NPC is immune to damage from a specific attacker based on flock or group affiliations.
3.  **Attack Scripting:** It provides a mechanism for an ordered list of attack overrides, allowing for scripted combat encounters or special abilities that deviate from standard AI behavior.

This class is a critical component in the server-side combat loop, acting as the single source of truth for an individual NPC's immediate combat readiness and damage immunities.

## Lifecycle & Ownership

-   **Creation:** An instance of CombatSupport is created exclusively during the construction of its parent NPCEntity. The instantiation is driven by a BuilderRole, which supplies the necessary configuration data (e.g., damage immunity rules) extracted from asset definitions.
-   **Scope:** The lifecycle of a CombatSupport object is identical to that of its parent NPCEntity. It persists as long as the NPC exists within the game world.
-   **Destruction:** The object has no explicit destruction or cleanup method. It becomes eligible for garbage collection when its parent NPCEntity is unloaded or destroyed, and all references to it are released.

## Internal State & Concurrency

-   **State:** This class is highly mutable and stateful. It maintains the current activeAttack, attack cooldown timers, friendly fire status, and a list of attackOverrides. This state is modified continuously throughout the NPC's combat routine. Configuration data, such as disableDamageGroups, is immutable after construction.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and operated by a single NPCEntity on the main server game thread. All methods, especially tick and setExecutingAttack, modify internal state without any synchronization mechanisms.

    **WARNING:** Accessing or modifying a CombatSupport instance from any thread other than the main server thread will lead to race conditions, inconsistent state, and server instability. All interactions must be synchronized with the server's primary tick loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isExecutingAttack() | boolean | O(1) | Returns true if the NPC is in an attack cooldown or an active attack animation. |
| tick(double dt) | void | O(1) | Advances internal timers. This must be called every server tick to manage attack cooldowns. |
| getCanCauseDamage(attackerRef, accessor) | boolean | O(N) | Evaluates if the parent NPC can be damaged by the specified attacker. Complexity is O(N) where N is the size of the entity group being checked. |
| setExecutingAttack(chain, damageFriendlies, pause) | void | O(1) | Sets the NPC's state to "attacking". This is the primary entry point for an AI to trigger a combat action. |
| addAttackOverride(attackSequence) | void | O(1) | Pushes a specific, named attack sequence to an internal queue for scripted execution. |
| getNextAttackOverride() | String | O(1) | Retrieves and cycles through the next available attack from the override queue. |
| clearAttackOverrides() | void | O(1) | Resets the attack override queue, returning the NPC to default AI behavior. |

## Integration Patterns

### Standard Usage

CombatSupport is managed by the NPC's active Role. The Role's update logic is responsible for ticking the component and using its state to inform AI decisions. When the AI decides to attack, it uses setExecutingAttack to initiate the action.

```java
// Example from within an NPC's Role update method

CombatSupport combat = this.parent.getRole().getCombatSupport();

// Must be ticked every frame to update cooldowns
combat.tick(deltaTime);

// AI decision logic
boolean canAttack = !combat.isExecutingAttack() && hasLineOfSight(target);
if (canAttack) {
    // Create the attack animation and logic chain
    InteractionChain attack = createMeleeAttackChain();
    
    // Commit the attack, starting a 2.0 second cooldown
    combat.setExecutingAttack(attack, false, 2.0);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new CombatSupport()`. The object will lack its parent NPCEntity and critical configuration from the builder, rendering it non-functional. It must be retrieved from the parent NPC's Role.
-   **External State Mutation:** Do not modify public fields directly if they were exposed. The internal state, such as attackPause, should only be manipulated via its public API, primarily setExecutingAttack and tick. Bypassing these methods will break the component's state machine.
-   **Skipping the Tick:** Failure to call the tick method in the main update loop will cause attack cooldowns to never expire, effectively locking the NPC out of combat permanently after its first attack.

## Data Pipeline

The CombatSupport component participates in two key server processes: attack execution and damage validation.

**Attack Execution Flow:**
This flow describes how an AI decision translates into a timed combat action.

> Flow:
> NPC AI System -> Decides to Attack -> Calls **CombatSupport.setExecutingAttack()** -> Internal state updated (attackPause > 0) -> Game Loop calls **CombatSupport.tick()** -> attackPause decrements -> AI queries **CombatSupport.isExecutingAttack()** -> Returns false when ready

**Damage Validation Flow:**
This flow describes how the server uses CombatSupport to prevent friendly fire or enforce group immunities.

> Flow:
> Damage Event Occurs -> Server Damage System -> Retrieves target's **CombatSupport** -> Calls **getCanCauseDamage()** -> Returns `false` if immune -> Damage is nullified by Server

