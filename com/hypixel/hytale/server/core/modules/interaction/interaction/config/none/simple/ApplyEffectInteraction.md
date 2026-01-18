---
description: Architectural reference for ApplyEffectInteraction
---

# ApplyEffectInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none.simple
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class ApplyEffectInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts

The ApplyEffectInteraction class is a data-driven, concrete implementation of an server-side interaction. It represents a single, atomic action: applying a specified status effect to a target entity. Architecturally, it functions as a **Strategy** within the broader server Interaction Module. Its behavior is not hard-coded but is instead defined by data loaded at runtime.

This class is not a service or a manager; it is a configuration object deserialized from asset files via its static **CODEC**. This design allows game designers to create complex item behaviors, world triggers, or NPC abilities by simply defining them in configuration files without requiring new Java code.

Its primary responsibility is to translate a high-level interaction event into a low-level command within the server's Entity-Component-System (ECS). It achieves this by using the provided InteractionContext to identify the target entity and then submitting a command to the **CommandBuffer**. This pattern ensures that all entity state mutations are deferred and executed deterministically at the end of the current server tick, preventing race conditions and maintaining world state integrity.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively by the Hytale codec system during server startup or asset loading. The static CODEC field defines the deserialization logic from a configuration file (e.g., HOCON or JSON). Direct instantiation is a critical anti-pattern.
-   **Scope:** An instance of ApplyEffectInteraction is a stateless, reusable configuration object. Its lifetime is tied to the asset registry. It persists as long as the asset defining it is loaded, typically for the entire server session. It is not tied to a specific player, entity, or session.
-   **Destruction:** The object is eligible for garbage collection when the server's AssetManager unloads the relevant asset packs, which usually occurs during a server shutdown.

## Internal State & Concurrency

-   **State:** The internal state, consisting of effectId and entityTarget, is configured once upon deserialization. Although the fields are not marked as final, they must be treated as **immutable** post-creation. Modifying this state at runtime will lead to undefined and inconsistent behavior across the server.
-   **Thread Safety:** This class is inherently thread-safe. Its immutable nature allows it to be safely read by multiple systems. The core logic within the firstRun method is designed to be executed on the main server thread for a specific world tick. It safely interacts with the ECS by queuing operations on the CommandBuffer, which is the designated mechanism for thread-safe entity state modification.

## API Surface

The public contract is primarily defined by its parent class, SimpleInstantInteraction. The following methods are the key implementation points.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the core interaction logic. Resolves the target entity and queues a command to apply the configured EntityEffect. Assumes asset and component lookups are constant time. |
| generatePacket() | Interaction | O(1) | Creates the network packet used to notify clients of this interaction. |
| configurePacket(packet) | void | O(1) | Populates the network packet with data, translating the string-based effectId into a more efficient numerical index for network transmission. |

## Integration Patterns

### Standard Usage

A developer or designer does not interact with this class directly in Java code. Instead, they define its properties within a higher-level asset, such as an item or a world trigger. The engine's Interaction Module is responsible for loading this configuration and invoking it.

A conceptual configuration might look like this:

```json
// Example: in a magic staff's item file
"onUseInteraction": {
  "type": "ApplyEffect",
  "EffectId": "hytale:poison_level_1",
  "Entity": "TARGET"
}
```

The server engine processes this configuration, instantiates an ApplyEffectInteraction object, and triggers its firstRun method when a player uses the magic staff.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new ApplyEffectInteraction()`. The object will be unconfigured and will cause NullPointerExceptions or other failures. All instances must be created via the asset loading pipeline using the class CODEC.
-   **Runtime State Mutation:** Do not get an instance of this class and attempt to change its effectId or entityTarget fields at runtime. These are meant to be immutable configuration values.
-   **External Invocation:** Do not call the firstRun method from outside the server's core Interaction Module. Doing so bypasses critical systems like cooldown handling, permission checks, and proper context setup.

## Data Pipeline

The flow of data and control for this interaction follows a standard server-authoritative ECS pattern.

> Flow:
> Player Input or Game Event -> Server Interaction Module -> **ApplyEffectInteraction.firstRun()** -> CommandBuffer.getComponent() -> EffectControllerComponent.addEffect() -> ECS Tick Processor -> World State Change -> Network Replication to Clients

