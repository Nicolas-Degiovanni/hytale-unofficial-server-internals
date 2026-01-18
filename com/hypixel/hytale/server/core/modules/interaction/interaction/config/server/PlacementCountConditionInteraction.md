---
description: Architectural reference for PlacementCountConditionInteraction
---

# PlacementCountConditionInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class PlacementCountConditionInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The PlacementCountConditionInteraction is a server-authoritative, conditional node within the Hytale Interaction System. It does not represent a player action but rather a logical gate that evaluates a world state before allowing an interaction sequence to proceed. Its primary function is to check if the total number of a specific block type placed in the world is above or below a configured threshold.

This class acts as a bridge between the abstract Interaction System and the concrete World State. It achieves this by querying a world-level resource, the BlockCounter, which is responsible for tracking persistent statistics about block manipulations.

Architecturally, instances of this class are not created programmatically. They are deserialized from game configuration files via the static CODEC field. This pattern allows game designers to construct complex, state-dependent interaction graphs without writing any Java code, simply by defining conditional checks like this one in asset files. The class is defined as a SimpleInstantInteraction, signifying that its logic executes completely within a single server tick.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale Codec system during server startup or when game configuration assets are loaded. The static CODEC field defines the mapping from configuration keys (e.g., "Block", "Value") to the object's internal fields.
-   **Scope:** An instance of this class is effectively a stateless, immutable template. It persists for as long as the configuration that defines it is loaded on the server. The logic of the class is executed within the transient scope of an InteractionContext, which is created each time an entity begins the interaction.
-   **Destruction:** The object is marked for garbage collection when the server unloads the corresponding game assets. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** Immutable. The fields blockType, value, and lessThan are set once during deserialization and are never modified thereafter. The class itself holds no runtime state; it only reads from external state sources.
-   **Thread Safety:** The object instance is inherently thread-safe due to its immutability. However, its methods operate on shared, mutable systems like the World and the InteractionContext. The parent Interaction System is responsible for ensuring that the firstRun method is not invoked concurrently for the same InteractionContext. The underlying BlockCounter resource is expected to be a thread-safe service.

## API Surface
The public contract is fulfilled by overriding methods from its parent class, SimpleInstantInteraction. Direct invocation is not an intended use case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | **Core Logic.** Executes the condition check. Reads from the global BlockCounter and mutates the passed InteractionContext state to either Finished or Failed. This is a protected method but represents the primary entry point from the interaction engine. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns WaitForDataFrom.Server, signaling to the engine that this interaction's logic is resolved entirely on the server and the client must await the outcome. |

## Integration Patterns

### Standard Usage
This class is not used by calling its methods directly in Java. Instead, it is defined declaratively within an interaction graph asset file (e.g., a JSON file). The server's Codec system will then instantiate and configure it automatically.

```json
// Example: Part of a quest interaction graph
{
  "type": "PlacementCountConditionInteraction",
  "Block": "hytale:stone_bricks",
  "Value": 100,
  "LessThan": false
}
```
In this example, the interaction will only succeed (transition to Finished) if the total number of placed hytale:stone_bricks in the world is greater than 100.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlacementCountConditionInteraction()`. This will create a misconfigured object that will cause NullPointerExceptions or other undefined behavior, as its internal fields will not be initialized. Always define it in a configuration file.
-   **Client-Side Execution:** Do not attempt to reference or use this class in client-side code. It has hard dependencies on server-only classes and resources, such as the world's ChunkStore and the BlockCounter.

## Data Pipeline
This component acts as a control-flow gate, not a data transformer. Its role is to read world state and direct the flow of the parent interaction.

> Flow:
> Interaction Engine begins sequence -> Invokes **PlacementCountConditionInteraction.firstRun** -> Reads from World.ChunkStore.BlockCounter -> Compares count to internal value -> Mutates InteractionContext.state -> Interaction Engine reads new state and either proceeds or terminates the sequence.

