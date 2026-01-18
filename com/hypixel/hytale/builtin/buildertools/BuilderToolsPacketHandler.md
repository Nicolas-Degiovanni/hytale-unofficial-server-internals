---
description: Architectural reference for BuilderToolsPacketHandler
---

# BuilderToolsPacketHandler

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolsPacketHandler implements SubPacketHandler {
```

## Architecture & Concepts
The BuilderToolsPacketHandler is a server-side network component that acts as the primary entry point for all client-initiated builder tool actions. It functions as a specialized **Network Boundary Layer**, translating raw data packets from a player's client into concrete, permission-checked commands and world modifications on the server.

Architecturally, this class is not a standalone service. It operates as a **pluggable sub-handler** registered with a parent IPacketHandler. This design decouples the core networking engine from game-specific logic, allowing features like the builder tools to be managed as an isolated module.

Its core responsibilities are:
1.  **Packet Routing:** In its `registerHandlers` method, it maps dozens of specific packet IDs to corresponding handler methods. If the BuilderToolsPlugin is disabled, it registers no-op handlers to gracefully discard incoming packets.
2.  **Permission Enforcement:** Nearly every handler method begins by invoking `hasPermission`. This class is the authoritative gatekeeper, ensuring that no builder tool operation can be performed without the appropriate player permissions. This is a critical security and gameplay boundary.
3.  **Context Resolution:** For each incoming packet, it resolves the associated PlayerRef, EntityStore, and World instance. This context is essential for executing any subsequent game logic.
4.  **Asynchronous Execution:** Handler methods do not perform world modifications directly. They wrap the logic in a lambda and submit it to the world's execution queue via `world.execute()`. This is a critical concurrency pattern that marshals work from the network I/O thread to the main server game thread, preventing race conditions and ensuring thread-safe access to game state.
5.  **Delegation to Core Logic:** This handler contains minimal business logic. Its primary role is to validate and delegate. Most complex operations, particularly those involving world edits, are forwarded to the `BuilderToolsPlugin`'s internal work queue. This further decouples network reception from the potentially long-running execution of tasks like pasting a large selection.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderToolsPacketHandler is created by its parent IPacketHandler, typically during the initialization of a player's network connection. It receives a reference to its parent handler via its constructor.
- **Scope:** The object's lifetime is strictly tied to its parent IPacketHandler. It persists for the duration of the player's session.
- **Destruction:** The object is eligible for garbage collection when the parent IPacketHandler is destroyed (e.g., on player disconnect). It has no explicit cleanup or `destroy` method.

## Internal State & Concurrency
- **State:** The BuilderToolsPacketHandler is effectively **stateless**. Its only instance field is a `final` reference to its parent IPacketHandler. All state required for its operations (e.g., player selection, tool settings) is retrieved from other systems like the Player component or the BuilderToolsPlugin on a per-request basis.
- **Thread Safety:** The class is inherently thread-safe due to its stateless design. However, its handler methods are designed to be invoked by a single network I/O thread. The critical concurrency management occurs when these methods use `world.execute()` to safely hand off state-modifying logic to the single-threaded game world, preventing data corruption and race conditions.

**WARNING:** Modifying this class to hold mutable state without proper synchronization would introduce severe concurrency bugs and is strongly discouraged.

## API Surface
The primary public contract is the `registerHandlers` method, which is part of the SubPacketHandler interface. The various `handle` methods are not intended for direct external invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerHandlers() | void | O(N) | Registers all builder tool packet handlers with the parent IPacketHandler. N is the number of packets. |
| handle(BuilderToolGeneralAction) | void | O(1) | Handles high-level actions like Undo, Redo, and Copy. Delegates to the BuilderToolsPlugin queue. |
| handle(BuilderToolOnUseInteraction) | void | O(1) | Handles primary brush usage. Delegates the core brush logic to the BuilderState via the plugin queue. |
| handle(BuilderToolSelectionTransform) | void | O(1) | Handles complex transformations of a player's selection. Queues a transform-and-paste operation. |
| handle(BuilderToolEntityAction) | void | O(1) | Handles actions on entities like Clone or Remove. Delegates to the server's CommandManager. |
| handle(BuilderToolArgUpdate) | void | O(1) | Updates server-side tool settings based on client UI changes. Delegates to BuilderToolsPlugin. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by feature developers. It is instantiated and managed by the server's core networking layer. The system integrates with it by registering it with a player's packet handler.

```java
// System-level code during player connection setup
// This is a conceptual example of how the system uses the class.
IPacketHandler playerPacketHandler = playerConnection.getPacketHandler();
BuilderToolsPacketHandler builderToolsHandler = new BuilderToolsPacketHandler(playerPacketHandler);

// The handler now registers its packet listeners.
builderToolsHandler.registerHandlers();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually create an instance of this class with `new BuilderToolsPacketHandler()`. Its lifecycle is managed by the network engine, which provides the required parent IPacketHandler dependency.
- **Direct Handler Invocation:** Never call a `handle` method directly from other server code. Doing so bypasses the network context resolution and thread-marshaling logic, leading to unpredictable behavior and likely exceptions. Packets are the only valid entry point.
- **Synchronous World Edits:** Do not modify a handler to perform world edits outside of a `world.execute()` block. This would execute logic on a network I/O thread, creating a critical race condition with the main game loop.

## Data Pipeline
The handler sits between the raw network layer and the core game logic, acting as a dispatcher. The flow for a typical brush action is as follows:

> Flow:
> Client Input -> BuilderToolOnUseInteraction Packet -> Server Network I/O Thread -> **BuilderToolsPacketHandler** -> World Execution Queue -> Game Thread -> BuilderToolsPlugin -> BuilderState.edit() -> World/Chunk Modification -> Client State Update Packet

