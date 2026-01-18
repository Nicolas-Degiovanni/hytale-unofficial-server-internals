---
description: Architectural reference for BasicAttackTargetCombatAction
---

# BasicAttackTargetCombatAction

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.evaluator.combatactions
**Type:** Transient

## Definition
```java
// Signature
public class BasicAttackTargetCombatAction extends CombatActionOption {
```

## Architecture & Concepts
The BasicAttackTargetCombatAction is a data-driven, concrete implementation of a combat option within the server-side NPC AI decision-making framework. It does not represent a complete attack sequence but rather a preparatory state transition. Its primary function is to configure an NPC entity to perform a basic melee or ranged attack in a subsequent AI state.

This class acts as a "leaf" in the NPC's behavior logic. It is selected by a higher-level system, the CombatActionEvaluator, based on situational scoring. Upon execution, it performs three critical setup tasks:
1.  **Equipment Setup:** It instructs the NPC to switch to a specific weapon and off-hand item by manipulating the NPC's inventory.
2.  **Range Configuration:** It writes the optimal engagement distance for the selected weapon into the NPC's ValueStore, a shared blackboard for AI components. This allows movement and targeting systems to act on a consistent goal.
3.  **State Finalization:** It signals to the parent CombatActionEvaluator that its setup action is complete, allowing the AI to proceed.

Crucially, this class is defined and configured through the Hytale Codec system. This means its parameters, such as which weapon slot to use, are specified in external asset files (e.g., JSON), not hardcoded in Java. This pattern allows designers to create a wide variety of NPC combat behaviors without modifying engine code.

### Lifecycle & Ownership
-   **Creation:** Instances are created by the Hytale `CODEC` deserialization pipeline when an NPC's behavior assets are loaded. It is never instantiated directly with the `new` keyword in game logic.
-   **Scope:** An instance persists as part of an NPC's loaded behavior definition. It is effectively a read-only configuration object for the lifetime of that NPC type's AI configuration.
-   **Destruction:** The object is eligible for garbage collection when the NPC's behavior definition is unloaded from memory, for instance, during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
-   **State:** The internal state, consisting of `weaponSlot` and `offhandSlot`, is set once during deserialization via the `CODEC`. The object is effectively immutable after this initial creation. The `execute` method modifies external state (the NPC, the ValueStore) but not its own internal fields.
-   **Thread Safety:** This class is not thread-safe. While its own state is immutable post-creation, the `execute` method is designed to operate exclusively on the main server thread for a given world region. It manipulates non-thread-safe components like the CommandBuffer and ArchetypeChunk. Invoking `execute` from any other thread will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is primarily for internal use by the NPC decision-making systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | void | O(1) | Configures the NPC's inventory and AI ValueStore for a basic attack. Must be called by the CombatActionEvaluator. |
| isBasicAttackAllowed(...) | boolean | O(1) | Predicate indicating that subsequent basic attack actions are permitted after this option runs. Always returns true. |
| cancelBasicAttackOnSelect() | boolean | O(1) | Predicate indicating if an in-progress basic attack should be cancelled. Always returns false. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code by developers. Instead, it is configured within an NPC's behavior asset file. The `CombatActionEvaluator` system will then load, select, and execute it automatically.

A conceptual configuration might look like this:

```json
// Example NPC Behavior Asset
{
  "id": "skeleton_archer_behavior",
  "combat": {
    "options": [
      {
        "type": "BasicAttackTargetCombatAction",
        "WeaponSlot": 1, // Use item in hotbar slot 1
        "OffhandSlot": -1 // No off-hand item
      }
    ]
  }
}
```

The system then invokes the action internally:

```java
// System-level code (conceptual)
// The evaluator selects this option based on its own internal logic.
CombatActionOption selectedOption = evaluator.selectBestOption();

// If the selected option is our BasicAttackTargetCombatAction,
// the system executes it on the target NPC entity.
if (selectedOption instanceof BasicAttackTargetCombatAction) {
    selectedOption.execute(entityIndex, chunk, commands, role, evaluator, valueStore);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BasicAttackTargetCombatAction()`. The object will be unconfigured and will cause NullPointerExceptions or incorrect behavior because the `weaponSlot` and `offhandSlot` fields will not be initialized.
-   **External State Modification:** Do not attempt to modify the `weaponSlot` or `offhandSlot` fields via reflection after the object has been created by the `CODEC`. This violates the data-driven design and can lead to unpredictable AI behavior.
-   **Asynchronous Execution:** Do not call the `execute` method from a separate thread or asynchronous task. All interactions with the entity component system must be synchronized with the main server tick.

## Data Pipeline
The execution of this action is a key step in the NPC combat data flow, translating a high-level decision into low-level state changes that drive other systems.

> Flow:
> NPC Behavior Asset File -> `CODEC` Deserializer -> **BasicAttackTargetCombatAction** Instance -> CombatActionEvaluator selects option -> `execute()` call -> Modifies `NPCEntity` Inventory & `ValueStore` -> Movement & Targeting Systems read `ValueStore` to position NPC for attack.

