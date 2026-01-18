---
description: Architectural reference for IncrementCooldownInteraction
---

# IncrementCooldownInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Data Object / Configuration Entity

## Definition
```java
// Signature
public class IncrementCooldownInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The IncrementCooldownInteraction class is a specific, data-driven command object within the server's Interaction System. It is not a persistent service but rather a definition for a single, atomic action: modifying an entity's cooldown timer.

Its primary role is to be deserialized from game configuration files (e.g., JSON assets) that define complex entity behaviors, such as item usage or ability activation. The static CODEC field is the cornerstone of this design, providing a schema for how the engine should read configuration data and instantiate this object.

Architecturally, this class encapsulates a piece of game logic that would otherwise be hardcoded. By defining the logic as data, designers can create and tweak game mechanics without requiring engine-level code changes. It acts as a bridge between the declarative configuration assets and the imperative state management of the CooldownHandler.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale serialization framework via the static `BuilderCodec<IncrementCooldownInteraction> CODEC`. A developer **must not** instantiate this class directly using the `new` keyword, as this bypasses essential configuration and initialization logic defined in the codec.
- **Scope:** Transient. An instance of this class is typically part of a larger interaction chain and exists only for the duration of that single interaction event. It is executed once via its `firstRun` method and holds no state beyond its initial configuration.
- **Destruction:** The object becomes eligible for garbage collection immediately after the interaction event completes and all references to it are dropped. It is not managed or pooled.

## Internal State & Concurrency
- **State:** The internal state (cooldown, cooldownTime, etc.) is effectively **immutable** post-deserialization. The fields are populated once by the CODEC during asset loading and are treated as read-only parameters for the `firstRun` execution logic.
- **Thread Safety:** This class is **not thread-safe** and is designed to be executed exclusively on the main server thread that manages game state. The `firstRun` method mutates the state of a shared `CooldownHandler`, and invoking it from a concurrent thread will lead to race conditions, data corruption, and server instability. All interactions with this object must be synchronized with the server's primary game loop.

## API Surface
The public API is minimal, reflecting its role as an internal command executed by the interaction system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Primary entry point. Executes the cooldown modification logic and immediately sets the interaction state to Finished. Throws NullPointerException if arguments are null. |
| generatePacket() | Interaction | O(1) | Factory method to create the corresponding network packet for client-side synchronization. |
| configurePacket(packet) | void | O(1) | Populates a given network packet with the configuration data from this object's fields. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in Java code. Instead, they define its properties within a game asset file. The engine then deserializes this data and executes the interaction. The following conceptual example shows how the *engine* might invoke a loaded interaction.

```java
// Engine-level code, not for typical game developers
// Assume 'interaction' was loaded from a JSON asset via the CODEC
IncrementCooldownInteraction interaction = loadInteractionFromAsset("my_ability.json");
InteractionContext context = createInteractionContextForPlayer(player);
CooldownHandler cooldownHandler = player.getCooldownHandler();

// The interaction system executes the loaded behavior
interaction.firstRun(InteractionType.PRIMARY, context, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new IncrementCooldownInteraction()`. This creates a useless, unconfigured object that will fail at runtime. Always define interactions in data files to be loaded by the engine's codec system.
- **State Mutation:** Do not attempt to modify the fields of this object after it has been created. It is a read-only data container for a single execution.
- **Asynchronous Execution:** Do not pass this object to another thread for execution. It must be run on the main server thread to safely interact with the `CooldownHandler`.

## Data Pipeline
The flow of data for this component begins with design-time configuration and ends with a change in server state and a corresponding client update.

> Flow:
> Game Asset (JSON) -> Engine Codec Deserializer -> **IncrementCooldownInteraction Instance** -> Interaction System Execution -> CooldownHandler State Mutation -> Network Packet Generation -> Client State Synchronization

