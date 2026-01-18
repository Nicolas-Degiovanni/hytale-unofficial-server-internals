---
description: Architectural reference for SendMessageInteraction
---

# SendMessageInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none.simple
**Type:** Transient

## Definition
```java
// Signature
public class SendMessageInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The SendMessageInteraction class is a concrete implementation within the server-side **Interaction System**. It represents a specific, self-contained action that sends a chat message to an entity. This class is a prime example of Hytale's data-driven design philosophy.

Its primary architectural role is to serve as a command object that is defined in external configuration files rather than being hardcoded in Java. The static **CODEC** field is the cornerstone of this design. It allows the game engine's asset loader to deserialize a block of configuration (e.g., JSON) into a fully-functional SendMessageInteraction instance at runtime.

This decouples high-level game design (e.g., "what happens when a player right-clicks this block") from low-level engine code. Designers can specify this interaction and its parameters (the message to send) in an asset file, and the Interaction Module will automatically execute its logic when the corresponding game event occurs.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale **Codec** system during server startup or when game assets are loaded. The `BuilderCodec` uses the no-argument constructor and reflection to populate the `message` and `key` fields from the asset data. Direct instantiation in game logic is a severe anti-pattern.
- **Scope:** An instance of SendMessageInteraction is stateless and its lifetime is tied to the parent object that defines it, such as a block or item definition. It persists in memory as part of that definition for the entire server session.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup when its parent asset definition is unloaded from memory, which typically only happens on server shutdown.

## Internal State & Concurrency
- **State:** The internal state, consisting of the `key` and `message` strings, is effectively **immutable** after being deserialized from configuration. It is designed to be read-only during gameplay.
- **Thread Safety:** This class is **not thread-safe** and must be used exclusively from the main server thread. The core logic in the `firstRun` method operates on an `InteractionContext` and `CommandBuffer`, which are thread-affine structures that manage access to the world state. Calling `firstRun` from an asynchronous task will lead to race conditions, data corruption, and server instability. All game state mutations are queued via the CommandBuffer for serialized execution at the end of the current tick.

## API Surface
The public API is minimal, as the class is primarily invoked by the Interaction System, not by general-purpose code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the message-sending logic. This is the sole entry point for the interaction's behavior and is called by the engine. Throws NullPointerException if context is invalid. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly in Java code. Instead, it is defined declaratively within a game asset file. The engine then loads this configuration and invokes the interaction when triggered.

A designer would define the interaction in a JSON asset like this:
```json
// Example: Part of a custom block's asset definition
{
  "id": "mymod:info_block",
  "components": {
    "interaction": {
      "type": "hytale:send_message",
      "Message": "This block was crafted by an ancient civilization."
    }
  }
}
```
The server's Interaction Module handles the discovery and execution of this configured object. No corresponding Java code is needed to use it.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new SendMessageInteraction()`. This bypasses the data-driven asset pipeline and creates brittle, hardcoded logic that is difficult to maintain and cannot be modified by designers.
- **State Mutation:** Do not attempt to get an instance of this class and modify its `message` or `key` fields at runtime. This can lead to unpredictable behavior, as the same instance may be shared by multiple game objects.
- **Asynchronous Execution:** Never invoke the `firstRun` method from a separate thread. All interactions must be processed on the main server thread to guarantee safe access to the game world.

## Data Pipeline
The flow of data from configuration to in-game effect is a one-way process managed by the engine.

> Flow:
> Game Asset (JSON File) -> Server Asset Loader -> **Codec Deserializer** -> In-Memory **SendMessageInteraction** Instance -> Game Event (e.g., PlayerUse) -> Interaction Module -> `firstRun` Execution -> `IMessageReceiver.sendMessage` -> Server Message Queue -> Network Packet to Client

