---
description: Architectural reference for CooldownConditionInteraction
---

# CooldownConditionInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class CooldownConditionInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The CooldownConditionInteraction is a server-side, data-driven component that functions as a conditional node within the Hytale Interaction System. It is not a long-lived service but rather a granular, single-purpose object representing one step in a potentially complex interaction sequence.

Its primary architectural role is to act as a gate. When an interaction is triggered, this component is executed to check a specific precondition: whether a named cooldown is currently active for the interacting entity. Based on the result, it directs the flow of the interaction by setting its state to either **Failed** or **Finished**.

This class is a concrete implementation of the engine's data-driven design philosophy. Game designers define interaction logic in external configuration files (e.g., JSON), and the static CODEC field is responsible for deserializing that data into a functional CooldownConditionInteraction instance at runtime. It serves as a bridge between the abstract Interaction System and the concrete CooldownHandler module, allowing for declarative, configuration-based game logic.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the engine's serialization system via the static **CODEC** field. This process occurs when the server loads game assets and configurations that define entity or item interactions. It is never instantiated directly in game logic code.
- **Scope:** Extremely short-lived and stateless. An instance exists only for the duration of a single interaction check. Its scope is confined to the execution of the `firstRun` method.
- **Destruction:** The object is eligible for garbage collection immediately after the interaction check is complete. It is not retained, cached, or managed by any system.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal state, specifically the `cooldown` identifier string, is set once during deserialization and is not modified during its lifecycle. The class itself holds no mutable game state.
- **Thread Safety:** The object is inherently thread-safe due to its immutability. However, its methods operate on the `InteractionContext`, which is a container for mutable and thread-sensitive game state.

**WARNING:** All method invocations on a CooldownConditionInteraction instance **must** be performed on the main server thread corresponding to the world in which the interaction occurs. Accessing the `InteractionContext` or `CooldownHandler` from other threads will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is minimal, designed for invocation by the parent Interaction System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the cooldown check. Mutates the `InteractionContext` state to Failed or Finished. Assumes CooldownHandler provides O(1) lookups. |
| generatePacket() | Interaction | O(1) | Creates a network packet to synchronize this interaction's details with the client. |
| configurePacket(packet) | void | O(1) | Populates the network packet with the specific cooldown ID being checked. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly through Java code. Instead, it is defined declaratively within a game data file as part of an interaction chain. The system then instantiates and executes it automatically.

**Conceptual Example (e.g., in a JSON asset):**
```json
{
  "onInteract": [
    {
      "type": "CooldownConditionInteraction",
      "Id": "special_ability_cooldown"
    },
    {
      "type": "ApplyEffectInteraction",
      "effect": "haste"
    }
  ]
}
```
In this example, the system first executes the CooldownConditionInteraction. If the `special_ability_cooldown` is active, the interaction state is set to **Failed**, and the subsequent ApplyEffectInteraction is never executed.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new CooldownConditionInteraction()`. This bypasses the CODEC and leaves the internal `cooldown` field null, which will cause a `NullPointerException` during the `firstRun` execution. The object is fundamentally incomplete without being configured by the deserializer.
- **Stateful Misuse:** Do not attempt to cache or reuse instances of this class. They are designed as single-use, transient objects. Reusing an instance across different interactions or ticks is not supported and has undefined behavior.

## Data Pipeline
The primary flow for this component is initiated by configuration data and results in a state change within the game world.

> Flow:
> Game Asset (JSON) -> Engine CODEC Deserializer -> **CooldownConditionInteraction Instance** -> Interaction System Execution -> `firstRun()` -> CooldownHandler Query -> `InteractionContext` State Mutation

