---
description: Architectural reference for BrushOriginArg
---

# BrushOriginArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient

## Definition
```java
// Signature
public class BrushOriginArg extends ToolArg<BrushOrigin> {
```

## Architecture & Concepts
The BrushOriginArg class is a specialized data container responsible for representing a builder tool's brush origin setting. It is a concrete implementation within a larger polymorphic argument-handling framework, inheriting from the generic ToolArg class.

Its primary architectural role is to bridge the gap between abstract configuration (e.g., a string in a config file) and a concrete, serializable network packet. It encapsulates the logic for parsing, validating, and encoding a single argument type—the BrushOrigin enum—for transmission to the client.

The system relies on a discriminated union pattern within the generic BuilderToolArg packet. BrushOriginArg uses the overridden **setupPacket** method to correctly populate this generic packet, setting the **argType** discriminator to BrushOrigin and attaching its own specialized data payload. This design allows the network layer to handle a single, consistent packet type while enabling extensible, type-safe argument handling on the server.

## Lifecycle & Ownership
-   **Creation:** Instances are created on-demand by the server's builder tool configuration system. This typically occurs when parsing a tool's definition from an asset file or when interpreting a player's in-game command. It is not managed by a dependency injection container or service registry.

-   **Scope:** Ephemeral and task-scoped. An instance of BrushOriginArg exists only for the duration of a single configuration or packet-building operation.

-   **Destruction:** The object is short-lived and becomes eligible for garbage collection as soon as the network packet it helped construct is created or the configuration context is discarded. There are no external references to manage.

## Internal State & Concurrency
-   **State:** Mutable. The class is a simple wrapper around a single **BrushOrigin** enum value. The state is held in the inherited **value** field. The static **CODEC** definition implies that this field is mutated directly during deserialization.

-   **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal locking or synchronization mechanisms. It is designed to be created, used, and discarded within a single thread of execution, typically the main server thread responsible for processing game logic or commands.

    **WARNING:** Sharing an instance of BrushOriginArg across multiple threads will result in race conditions and undefined behavior. Do not cache or reuse instances in a concurrent environment.

## API Surface
The public API is minimal, focusing exclusively on data conversion and packet preparation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | BrushOrigin | O(1) | Parses a case-sensitive string into a BrushOrigin enum. Throws IllegalArgumentException if the string is not a valid enum constant. |
| toBrushOriginArgPacket() | BuilderToolBrushOriginArg | O(1) | Creates a specialized, type-safe network packet containing the internal BrushOrigin value. |
| setupPacket(BuilderToolArg packet) | void | O(1) | Configures a generic packet wrapper, setting its type discriminator and attaching the specific data payload from this argument. |

## Integration Patterns

### Standard Usage
The primary use case is to populate a generic network packet during the processing of a builder tool command or action. The instance acts as a strategy for configuring the packet correctly.

```java
// How a developer should normally use this
// 1. Create the specific argument type, perhaps from player input.
BrushOriginArg originArg = new BrushOriginArg(BrushOrigin.PLAYER);

// 2. Create the generic packet container.
BuilderToolArg genericPacket = new BuilderToolArg();

// 3. The BrushOriginArg instance correctly configures the generic packet.
originArg.setupPacket(genericPacket);

// 4. The genericPacket is now ready for the network layer.
//    It contains argType = BrushOrigin and the specific brushOriginArg payload.
networkManager.send(player, genericPacket);
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not modify and reuse a BrushOriginArg instance for multiple, distinct operations. Its lifecycle is intended to be ephemeral. Create a new instance for each packet or configuration.

-   **Direct Packet Manipulation:** Avoid manually setting fields on the generic BuilderToolArg packet. The `setupPacket` method is the authoritative source for configuration and ensures that the type discriminator and data payload are consistent. Manually setting these fields can lead to serialization errors or client-side desynchronization.

    ```java
    // BAD: Bypasses the type-safe setup logic.
    BuilderToolArg packet = new BuilderToolArg();
    packet.argType = BuilderToolArgType.BrushOrigin;
    packet.brushOriginArg = new BuilderToolBrushOriginArg(BrushOrigin.PLAYER); // Prone to error
    ```

## Data Pipeline
BrushOriginArg serves as a critical transformation step in the server's data flow for builder tools, converting high-level configuration into a low-level network representation.

> Flow:
> Server-Side Configuration (Asset File or Command) -> `fromString` -> **BrushOriginArg** Instance -> `setupPacket` -> `BuilderToolArg` Packet -> Network Serialization Layer -> Client

---

