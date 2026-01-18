---
description: Architectural reference for LANDiscoveryCommand
---

# LANDiscoveryCommand

**Package:** com.hypixel.hytale.builtin.landiscovery
**Type:** Transient Handler

## Definition
```java
// Signature
public class LANDiscoveryCommand extends CommandBase {
```

## Architecture & Concepts
The LANDiscoveryCommand class is a *Command Object* that serves as the user-facing entry point for controlling the server's Local Area Network (LAN) discovery broadcasting feature. It is not responsible for the implementation of network broadcasting itself; rather, it acts as a thin controller layer that translates user commands into state changes within the core `LANDiscoveryPlugin`.

This class strictly adheres to the Command pattern, encapsulating a request (enable, disable, or toggle LAN discovery) as an object. Its primary role is to interface with the server's command processing system, parse user-provided arguments, and delegate the core logic to the appropriate service, in this case, the `LANDiscoveryPlugin`. This design decouples the user interface (the command) from the business logic (the network broadcasting thread), promoting modularity and clear separation of concerns.

## Lifecycle & Ownership
- **Creation:** A single instance of LANDiscoveryCommand is instantiated by the server's command registration system during the bootstrap phase. The system scans for classes extending `CommandBase` and registers them for runtime invocation.
- **Scope:** The object instance persists for the entire duration of the server session. It is held as a reference within a central command registry or manager.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its internal fields, such as `MESSAGE_IO_LAN_DISCOVERY_ENABLED` and `enabledArg`, are final and are initialized at construction time. All mutable state related to whether LAN discovery is active is owned and managed exclusively by the `LANDiscoveryPlugin` singleton.
- **Thread Safety:** The `executeSync` method is guaranteed by the command system to be invoked on the main server thread. This is a critical design constraint to prevent race conditions when interacting with game-state-aware services like the `LANDiscoveryPlugin`. Direct invocation from asynchronous threads is unsupported and will lead to unpredictable server behavior. The class's immutable internal state makes it inherently thread-safe, but its *usage* is restricted to the main thread.

## API Surface
The public contract is defined by its inheritance from `CommandBase` and is intended for framework use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeSync(CommandContext) | protected void | O(1) | Framework entry point. Toggles or sets the LAN discovery state by delegating to `LANDiscoveryPlugin`. Sends feedback to the command issuer. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is invoked by the server's command handler in response to user input from the console or an in-game entity with sufficient permissions.

A server administrator would execute the command as follows:

```sh
# Toggles the current LAN discovery state
landiscovery

# Explicitly enables LAN discovery
landiscovery true

# Explicitly disables LAN discovery
landiscovery false
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create an instance using `new LANDiscoveryCommand()`. The command system manages the lifecycle of this object. Manually creating an instance serves no purpose as it will not be registered to handle user input.
- **Direct Invocation:** Never call the `executeSync` method directly. Doing so bypasses the command system's permission checks, context building, and thread safety guarantees. To control LAN discovery programmatically, use the `LANDiscoveryPlugin` API.

## Data Pipeline
LANDiscoveryCommand acts as a control-flow trigger, not a data processor. Its execution path translates a user action into a system state change.

> Flow:
> User Input (`/landiscovery true`) -> Server Command Parser -> **LANDiscoveryCommand.executeSync()** -> `LANDiscoveryPlugin.setLANDiscoveryEnabled(true)` -> (System spawns or terminates the UDP broadcast thread)

