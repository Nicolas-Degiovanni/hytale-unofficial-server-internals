---
description: Architectural reference for ChainFlagInteraction
---

# ChainFlagInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Configuration Model / Transient

## Definition
```java
// Signature
public class ChainFlagInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The ChainFlagInteraction class is a data-driven configuration object, not a traditional service or manager. It represents a specific, concrete action within the server's Interaction System. Its primary architectural role is to act as a server-side factory for a network packet destined for the client.

This class embodies a core design principle of the Hytale engine: defining game logic and behavior through external data (e.g., JSON assets) rather than hard-coded Java. An instance of ChainFlagInteraction is deserialized from a configuration file at runtime using its static CODEC.

Critically, this interaction performs **no server-side logic** when triggered. The `firstRun` method is intentionally empty. Its sole responsibility is to instantiate and populate a `com.hypixel.hytale.protocol.ChainFlagInteraction` packet with its configured state (a `chainId` and a `flag`). This packet is then dispatched to the client, which is responsible for interpreting the flag and executing the corresponding client-side logic, such as updating a UI or triggering a visual effect.

This pattern effectively decouples server-side triggers from client-side presentation and logic, allowing content designers to orchestrate complex client behaviors without modifying server code.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the engine's serialization system via the static `CODEC` field. This process occurs when the server loads game assets and configurations, typically during startup or when a new zone is loaded. Direct instantiation is an anti-pattern and will lead to an unconfigured, non-functional object.
-   **Scope:** The lifetime of a ChainFlagInteraction instance is tied to the game object or configuration it is part of. It persists in memory as a stateless, immutable data container for as long as its parent configuration is active.
-   **Destruction:** The object is eligible for garbage collection when the server unloads the associated assets or game objects. There is no manual destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state consists of two fields: `chainId` and `flag`. This state is **effectively immutable**. It is populated once during deserialization by the CODEC and is not designed to be modified thereafter.
-   **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable state, instances can be safely accessed and used by multiple threads simultaneously without the need for external locking or synchronization. The methods for packet generation are pure functions of the initial state.

## API Surface
The public contract is primarily defined by the `SimpleInstantInteraction` parent class. The overridden methods are specialized for packet generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Entry point when the interaction is triggered. Intentionally empty; performs no server-side action. |
| generatePacket() | Interaction | O(1) | Factory method. Instantiates a new `com.hypixel.hytale.protocol.ChainFlagInteraction` packet. |
| configurePacket(packet) | void | O(1) | Populates the provided network packet with the `chainId` and `flag` from this object's internal state. |

## Integration Patterns

### Standard Usage
A developer or content designer does not interact with this class directly in Java. Instead, they define it declaratively within a game asset file (e.g., JSON). The server's Interaction Module is responsible for invoking it.

The following is a conceptual representation of how the system uses the object after it has been loaded from a configuration.

```java
// Conceptual example from within the Interaction Module
// An instance of ChainFlagInteraction is retrieved from a component's configuration.
SimpleInstantInteraction interaction = entity.getInteraction("on_use");

// The module triggers the interaction. This will internally call firstRun,
// generatePacket, and configurePacket to send the data to the client.
interaction.run(interactionContext);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ChainFlagInteraction()`. The object will be uninitialized and will cause NullPointerExceptions when its methods are called. Always define interactions in data files to be loaded by the server's CODEC system.
-   **State Mutation:** Do not attempt to modify the `chainId` or `flag` fields after the object has been created. This violates the immutable design contract and can lead to unpredictable behavior.
-   **Adding Server Logic:** Do not extend this class to add logic to the `firstRun` method. The purpose of this specific class is to be a client-bound packet factory. If server-side logic is required, a different base interaction class must be used.

## Data Pipeline
The primary function of this class is to transform a server-side game event into a client-bound network packet.

> Flow:
> Game Event (e.g., Player Use) -> Interaction Module finds configured `ChainFlagInteraction` -> **ChainFlagInteraction.generatePacket()** -> **ChainFlagInteraction.configurePacket()** -> Server Network Layer -> Client Network Layer -> Client-Side Interaction System consumes `chainId` and `flag`

