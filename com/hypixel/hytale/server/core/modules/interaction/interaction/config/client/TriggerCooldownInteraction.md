---
description: Architectural reference for TriggerCooldownInteraction
---

# TriggerCooldownInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class TriggerCooldownInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts

The TriggerCooldownInteraction is a specialized, server-side component within the Hytale Interaction System. It functions as a *meta-interaction*; its primary purpose is not to manifest a direct, visible effect in the game world (such as dealing damage or opening a door), but to manipulate the state of the server's cooldown mechanics for a given player.

This class is designed to be configured and loaded from game asset files, not instantiated directly in code. Its static CODEC field facilitates this data-driven approach, allowing designers to embed cooldown triggers within complex sequences of actions, known as an InteractionChain.

Architecturally, it acts as a declarative instruction. When executed, it resolves the definitive cooldown parameters by consulting a strict hierarchy of sources:
1.  **Local Override:** A specific `InteractionCooldown` object defined directly within this interaction's configuration.
2.  **Chain Root:** The default `InteractionCooldown` defined on the `RootInteraction` of the current `InteractionChain`.
3.  **Global Identifier:** A fallback to the ID of the `RootInteraction` itself.

This resolved configuration is then used to command the `CooldownHandler` to apply the cooldown, effectively starting the timer and consuming a charge. This decouples the *act* of triggering a cooldown from other game logic, promoting a highly composable and maintainable interaction system.

## Lifecycle & Ownership

-   **Creation:** Instances are deserialized from game asset files (e.g., JSON or HOCON) by the server's asset loading pipeline using the provided `BuilderCodec`. It is never instantiated via a direct `new` call in gameplay logic. It exists as part of a larger, in-memory graph of interaction configurations.
-   **Scope:** The object's lifetime is tied to the server's asset cache. It persists as a read-only template for as long as its parent `RootInteraction` configuration is loaded.
-   **Destruction:** The object is eligible for garbage collection when the server unloads the asset set it belongs to, typically during a full server shutdown or a dynamic asset reload.

## Internal State & Concurrency

-   **State:** The class is effectively **immutable** after its initial creation by the codec. Its internal `cooldown` field is a configuration parameter set at load time and is not intended to be mutated during the game loop.
-   **Thread Safety:** This class is **thread-safe for read operations**. Because its internal state is immutable post-creation, a single instance can be safely referenced by multiple threads processing different player interactions. However, its primary method, `firstRun`, modifies the state of a passed-in `CooldownHandler`. The `CooldownHandler` itself is the component responsible for ensuring thread safety and managing concurrent access to player cooldown data. This class safely delegates all state mutation to that external, thread-safe system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | The primary entry point. Resolves cooldown parameters and instructs the CooldownHandler to apply the cooldown and deduct a charge. This is the core operational method. |
| generatePacket() | Interaction | O(1) | Creates a network packet to inform the client that a cooldown has been triggered. |

## Integration Patterns

### Standard Usage

A developer or designer does not interact with this class directly via Java code. Instead, it is defined declaratively within an interaction asset file. The system invokes it automatically when processing an `InteractionChain`. The following code is a conceptual representation of how the engine executes this interaction.

```java
// This class is not invoked directly by developers.
// It is executed by the InteractionManager when processing an InteractionChain.
// The following is a conceptual representation of the engine's internal call.

InteractionContext context = ...; // Provided by the engine for the current interaction
CooldownHandler cooldownHandler = context.getPlayer().getCooldownHandler();

// The engine retrieves the current interaction step, which is a TriggerCooldownInteraction instance
TriggerCooldownInteraction interaction = (TriggerCooldownInteraction) context.getCurrentInteraction();

// The engine executes the interaction's logic
interaction.firstRun(InteractionType.PRIMARY, context, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new TriggerCooldownInteraction()`. The object will be unconfigured and will cause `NullPointerException` or assertion failures when used. Always define it within game data assets to be loaded by the server's codec system.
-   **State Mutation:** Do not attempt to get and modify the internal `cooldown` field at runtime. This is static configuration data. Modifying it would be a non-thread-safe operation with unpredictable side effects for all subsequent interactions using this template.
-   **Misusing as a State Tracker:** Do not hold a reference to this object to check the status of a cooldown. This class only *triggers* the cooldown. The authoritative state of an active cooldown is managed exclusively by the `CooldownHandler`.

## Data Pipeline

The data flow for this component involves both server-side state changes and network communication to the client.

> Flow:
> Player Input -> Server-Side InteractionManager -> InteractionChain Execution -> **TriggerCooldownInteraction.firstRun()** -> CooldownHandler (Server State Change) -> **TriggerCooldownInteraction.generatePacket()** -> Network Layer -> Client UI Update

