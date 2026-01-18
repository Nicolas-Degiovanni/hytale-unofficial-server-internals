---
description: Architectural reference for ToggleGliderInteraction
---

# ToggleGliderInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Configuration Object

## Definition
```java
// Signature
public class ToggleGliderInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The ToggleGliderInteraction class is a server-side configuration object that represents a client-authoritative action. It does **not** contain the logic for toggling a glider. Instead, its sole responsibility is to act as a factory for the network packet that instructs the client to perform the glider toggle action.

This class is a component of the server's data-driven Interaction System. It allows server administrators to define that a "toggle glider" action exists and can be triggered, but delegates the implementation of the visual and physical effects to the client. This design decouples server-side game rules (what actions are possible) from client-side presentation (how those actions manifest).

By extending SimpleInstantInteraction, it inherits the contract for interactions that occur immediately without a duration or channeling period. The empty implementation of its `firstRun` method signifies that this interaction requires no initial state change or validation on the server; it is a pure "fire-and-forget" signal to the client.

## Lifecycle & Ownership
- **Creation:** Instances are not created manually via the `new` keyword. They are deserialized from server configuration files (e.g., JSON) at startup by a higher-level service using the public static `CODEC` field. This is a core part of the engine's data-driven architecture.
- **Scope:** An instance of ToggleGliderInteraction is effectively a singleton for its configured type. It is loaded once and held in an engine-level registry, such as an InteractionModule, for the entire duration of the server session.
- **Destruction:** The object is eligible for garbage collection only when the server shuts down or performs a full reload of its interaction configurations.

## Internal State & Concurrency
- **State:** **Immutable**. This class contains no instance fields and its behavior is fixed upon creation. It is a stateless factory.
- **Thread Safety:** **Fully Thread-Safe**. Due to its immutable nature, an instance can be safely shared and accessed by multiple threads (e.g., different player processing threads) without any need for locks or synchronization. The `generatePacket` method produces a new, distinct packet object on each invocation.

## API Surface
The primary contract is defined by its parent, SimpleInstantInteraction. The methods below are the key implementations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(...) | void | O(1) | No-operation. This interaction is entirely client-side and requires no server-side state change upon execution. |
| generatePacket() | Interaction | O(1) | **CRITICAL:** Factory method that creates the network packet to be sent to the client, instructing it to toggle the glider. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most game logic developers. It is invoked internally by the server's core Interaction System after a trigger condition is met.

```java
// Pseudo-code for internal engine usage
// 1. The configuration is retrieved from a registry
InteractionConfiguration config = interactionRegistry.get("player.toggle_glider");

// 2. The system invokes the packet factory method
if (config instanceof ToggleGliderInteraction) {
    Interaction packet = ((ToggleGliderInteraction) config).generatePacket();

    // 3. The packet is dispatched to the relevant client
    player.getNetworkConnection().send(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new ToggleGliderInteraction()`. The system relies on loading these objects from configuration files via the `CODEC`. Manual instantiation bypasses the data-driven architecture and will likely lead to an unregistered, non-functional interaction.
- **Adding Server-Side Logic:** Do not subclass this to add server-side effects. This class is strictly for generating a client-bound packet. Server-side logic for an interaction must be implemented in a different class structure designed for that purpose.

## Data Pipeline
The primary function of this class is to serve as a single step in the data flow from server configuration to client action.

> Flow:
> Server Configuration File (JSON) -> `BuilderCodec` Deserializer -> **ToggleGliderInteraction Instance** -> Interaction System Trigger -> `generatePacket()` -> `com.hypixel.hytale.protocol.ToggleGliderInteraction` Packet -> Network Layer -> Client Game Engine

