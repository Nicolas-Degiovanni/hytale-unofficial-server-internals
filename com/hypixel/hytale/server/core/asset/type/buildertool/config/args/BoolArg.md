---
description: Architectural reference for BoolArg
---

# BoolArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient

## Definition
```java
// Signature
public class BoolArg extends ToolArg<Boolean> {
```

## Architecture & Concepts
The BoolArg class is a server-side data model representing a single, boolean-typed argument for a Builder Tool. It serves as a concrete implementation within a larger, polymorphic system for tool configuration, where the generic ToolArg class defines the base contract.

Its primary responsibilities are threefold:
1.  **Data Representation:** To hold the state of a boolean argument, such as a "Snap to Grid" toggle or an "Overwrite" flag.
2.  **Serialization Contract:** To define its own data-driven configuration via the static CODEC field. The Hytale engine uses this codec to deserialize BoolArg instances from asset files (e.g., JSON) when loading a Builder Tool's definition.
3.  **Network Translation:** To convert its server-side state into a specific network packet, BuilderToolBoolArg, for transmission to the client. This ensures the client-side UI can accurately represent the tool's configurable options.

This class is a fundamental link between static asset configuration on disk and the dynamic, networked state of in-game tools.

### Lifecycle & Ownership
-   **Creation:** BoolArg instances are primarily created by the engine's serialization system, specifically the BuilderCodec, when a Builder Tool asset is loaded from storage. They can also be instantiated programmatically by server logic when dynamically constructing a tool.
-   **Scope:** The lifetime of a BoolArg is strictly bound to its parent Builder Tool configuration object. It is a value object and does not exist independently.
-   **Destruction:** The object is marked for garbage collection when its parent configuration is unloaded or goes out of scope. It requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The class is mutable. Its internal boolean state is held in the inherited *value* field, which is populated during deserialization or construction.
-   **Thread Safety:** This class is **not thread-safe**. As a simple data container, it possesses no internal locking mechanisms. It is designed to be created and configured during the server's single-threaded asset loading phase. Any subsequent mutation must be performed on the main server thread to prevent data races.

## API Surface
The public API is focused on serialization and network packet conversion.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | Boolean | O(1) | Parses a boolean value from a string representation. Used for command-line or text-based tool invocation. |
| toBoolArgPacket() | BuilderToolBoolArg | O(1) | Creates a new, specialized network packet containing the boolean value. |
| setupPacket(BuilderToolArg packet) | void | O(1) | Configures a generic parent packet, setting its type discriminator to Bool and attaching the specialized payload from toBoolArgPacket. |

## Integration Patterns

### Standard Usage
The BoolArg is not typically manipulated directly. Instead, the engine's Builder Tool system interacts with it through the generic ToolArg interface, primarily during the process of synchronizing a tool's state with a client.

```java
// Hypothetical engine code to prepare a tool for network sync
BuilderToolArg genericPacket = new BuilderToolArg();
BoolArg boolArg = loadedTool.getArgument("overwrite"); // Assumes a lookup

// The engine calls the polymorphic setup method
// This internally calls boolArg.setupPacket(genericPacket)
boolArg.preparePacket(genericPacket); 

// The genericPacket is now configured with boolean-specific data
// and is ready to be sent to the client.
networkManager.send(player, genericPacket);
```

### Anti-Patterns (Do NOT do this)
-   **State Desynchronization:** Modifying the internal *value* of a BoolArg after its parent tool has been synchronized with the client, without triggering a new network update. This will cause the server's logic and the client's UI to diverge.
-   **Incorrect Packet Handling:** Calling `toBoolArgPacket` but failing to embed the resulting object into the parent `BuilderToolArg` via `setupPacket`. The client will receive a generic packet with a null payload, likely causing a client-side error.

## Data Pipeline
The BoolArg acts as a conduit for configuration data, transforming it from a static asset definition into a live network object.

> Flow:
> Builder Tool JSON Asset -> **BuilderCodec** -> **BoolArg Instance (Server Memory)** -> `setupPacket` call -> BuilderToolArg (Network Packet) -> TCP/IP Stack -> Client Game Instance

