---
description: Architectural reference for SerialInteraction
---

# SerialInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Transient / Data Object

## Definition
```java
// Signature
public class SerialInteraction extends Interaction {
```

## Architecture & Concepts

The SerialInteraction class is a fundamental implementation of the **Composite Pattern** within the server's interaction module. It does not define a unique action itself; instead, it functions as a container that executes a sequence of other defined interactions in a specified order. This allows game designers to chain together simpler interactions (e.g., play a sound, apply a force, grant an item) to create complex, multi-stage behaviors without writing new code.

Architecturally, SerialInteraction is a passive data structure that represents a configured sequence. It is not an active, stateful object that receives updates in the main game loop. This is explicitly enforced by the `tick0` and `simulateTick0` methods, which throw an IllegalStateException if called.

Its primary roles are fulfilled during two key engine phases:

1.  **Compilation:** During server initialization or world loading, the `compile` method is invoked. It traverses its list of child interactions and instructs an OperationsBuilder to construct a low-level, optimized sequence of executable commands. This pre-processing step converts the high-level configuration into a performant runtime representation.
2.  **Traversal & Data Collection:** The `walk` method is used by higher-level systems, like the InteractionManager, to traverse the interaction graph. This can be used for validation, data gathering, or conditional checks before an interaction is executed. The method short-circuits, stopping its traversal as soon as any child interaction returns true.

This class serves as a critical link between declarative game data (often defined in JSON or HOCON files) and the server's imperative execution logic.

### Lifecycle & Ownership
-   **Creation:** An instance of SerialInteraction is never created directly with the `new` keyword. It is instantiated exclusively by the Hytale codec system, specifically its static `CODEC` field, during the deserialization of game asset files.
-   **Scope:** The object is immutable after creation. Its lifetime is bound to the asset cache. It persists as long as the corresponding game configuration is loaded, typically for the duration of a server session or until a world is unloaded.
-   **Destruction:** The object is marked for garbage collection when the Interaction asset registry is cleared, for example, on server shutdown.

## Internal State & Concurrency
-   **State:** Immutable. The core state, the `interactions` array of strings, is populated by the codec upon creation and is never modified thereafter. This design guarantees that a given SerialInteraction definition is consistent and predictable throughout its lifetime.
-   **Thread Safety:** Inherently thread-safe. Due to its immutable nature, a SerialInteraction instance can be safely read by multiple threads simultaneously without locks or other synchronization primitives. The `compile` and `walk` methods are pure functions with respect to the object's state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| walk(collector, context) | boolean | O(N) | Traverses the child interactions, invoking `InteractionManager.walkInteraction` on each. N is the number of child interactions. Short-circuits and returns true if any child walk returns true. |
| compile(builder) | void | O(N) | Recursively compiles each child interaction into a low-level operation list using the provided OperationsBuilder. |
| generatePacket() | Interaction | O(1) | Creates the corresponding network packet used to represent this interaction on the client. |
| configurePacket(packet) | void | O(N) | Populates a network packet with the integer IDs of the child interactions, optimizing for network bandwidth. |

## Integration Patterns

### Standard Usage

A developer or designer does not interact with this class directly in code. Instead, they define its behavior declaratively in an asset file. The engine's InteractionManager then uses the instance created by the codec system.

A conceptual asset definition might look like this:

```json
// in some_interaction.json
{
    "type": "SerialInteraction",
    "interactions": [
        "hytale:play_sound_effect_wood_click",
        "hytale:apply_knockback_small",
        "hytale:consume_item_in_hand"
    ]
}
```

The engine would then load this asset and might use it as follows:

```java
// Engine-level code, not typical user code
Interaction interaction = Interaction.getInteractionOrUnknown("hytale:some_interaction");
OperationsBuilder builder = new OperationsBuilder();
interaction.compile(builder); // Compiles the SerialInteraction and its children
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SerialInteraction()`. The object will be in an invalid state as its internal `interactions` array will be null. Always define interactions in asset files and retrieve them via the InteractionManager.
-   **Calling Tick Methods:** Do not call `tick0` or `simulateTick0`. These methods are intentionally designed to throw an exception. The logic of a SerialInteraction is expressed through compilation and traversal, not per-frame ticks.
-   **Expecting Remote Sync:** Do not rely on this interaction to synchronize complex state. Its `needsRemoteSync` method returns false, indicating its behavior is determined by its static definition, which is assumed to already exist on the client.

## Data Pipeline

The SerialInteraction class participates in two primary data flows: the asset compilation pipeline and the network serialization pipeline.

**Asset Compilation Pipeline**
> Flow:
> JSON Asset File -> Hytale Codec System -> **SerialInteraction Instance** -> InteractionManager.compile() -> OperationsBuilder -> Finalized Operation List

**Network Serialization Pipeline**
> Flow:
> **SerialInteraction Instance** -> configurePacket() -> `protocol.SerialInteraction` Packet (with integer IDs) -> Network Encoder -> TCP/UDP Packet -> Client

