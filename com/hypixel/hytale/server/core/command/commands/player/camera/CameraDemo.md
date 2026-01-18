---
description: Architectural reference for CameraDemo
---

# CameraDemo

**Package:** com.hypixel.hytale.server.core.command.commands.player.camera
**Type:** Singleton

## Definition
```java
// Signature
public class CameraDemo {
```

## Architecture & Concepts
The CameraDemo class implements a server-authoritative, non-standard gameplay mode. It is not a core engine system but rather a feature module, likely intended for development, testing, or special events. When activated, it transforms the player experience from standard gameplay into a top-down, real-time strategy (RTS) or world-editor perspective.

Architecturally, it functions as a global state machine with two states: *active* and *inactive*.

When **active**, CameraDemo operates by:
1.  **Intercepting Player Input:** It registers high-priority listeners for player interaction events, such as PlayerMouseButtonEvent and PlayerInteractEvent. It cancels the standard behavior of these events to prevent normal gameplay actions.
2.  **Overriding Client Cameras:** It sends a specially configured SetServerCamera packet to every connecting and currently connected player. This packet instructs the client to detach its camera from the player entity and adopt a custom view defined by the server.
3.  **Implementing Custom Logic:** The intercepted input events are repurposed to drive a custom world-editing feature. Mouse clicks are translated into block placement and removal operations directly within the game world.

This class is a prime example of a server-side module that completely dictates a client's control scheme and visual perspective, demonstrating the power of the server-authoritative camera and input systems.

## Lifecycle & Ownership
-   **Creation:** The singleton INSTANCE is instantiated eagerly by the JVM during class loading. This occurs once when the CameraDemo class is first referenced.
-   **Scope:** The CameraDemo object exists for the entire lifetime of the server process. However, its *functional* lifecycle is explicitly managed via the activate and deactivate methods. The system is dormant until activate is called.
-   **Destruction:** The object is garbage collected when the server shuts down. The deactivate method provides the explicit cleanup logic, which is critical for restoring default player behavior. It unregisters all event listeners and sends packets to reset player cameras to their default state.

**WARNING:** Failure to call deactivate before discarding the system will leave players in a broken state, with custom cameras and hijacked input handlers persisting indefinitely.

## Internal State & Concurrency
-   **State:** The class is stateful. Its primary state is the mutable boolean flag, isActive, which dictates whether the custom game mode is currently enforced. It also holds a final reference to a pre-configured ServerCameraSettings object that defines the "demo" camera view.
-   **Thread Safety:** This class is **not** thread-safe by design and expects to be managed from a single, serialized context, such as the main server thread or a command processing queue.
    -   The isActive flag is not volatile and is not protected by any locks. Concurrent calls to activate and deactivate will lead to race conditions and unpredictable behavior.
    -   The internal EventRegistry uses a CopyOnWriteArrayList, making event registration and unregistration thread-safe.
    -   Event handler methods like onPlayerMouseButton perform direct world modification. These operations are fundamentally unsafe to perform outside the main server tick thread. The design relies on the server's global EventBus to dispatch these events on the correct thread, which is a standard and safe engine pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | CameraDemo | O(1) | Public static accessor for the singleton instance. |
| activate() | void | O(N) | Enables the demo mode. Registers event listeners and forces custom cameras on all N current players. |
| deactivate() | void | O(N) | Disables the demo mode. Unregisters listeners and resets cameras for all N current players. |

## Integration Patterns

### Standard Usage
The system should be treated as a global feature flag, activated and deactivated through the singleton instance. This is typically done in response to a server command or another system-level trigger.

```java
// To enable the camera demo mode for all players
CameraDemo.INSTANCE.activate();

// To disable the mode and restore normal gameplay
CameraDemo.INSTANCE.deactivate();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new CameraDemo()`. While the default constructor is public, this violates the singleton pattern. It will create an orphaned object that has no connection to the static INSTANCE used by server systems. Always use `CameraDemo.INSTANCE`.
-   **State Unawareness:** Do not call activate or deactivate without checking the system's current state if it matters for your logic. Calling activate multiple times is a safe no-op but is inefficient.
-   **Incomplete Lifecycle:** Never call activate without a corresponding call to deactivate in your code's logic. Leaving this mode active permanently will break standard gameplay mechanics.

## Data Pipeline
CameraDemo acts as a consumer of server events and a producer of network packets and world state changes.

**Pipeline 1: Player Connection / Mode Activation**
> Flow:
> PlayerConnectEvent -> **CameraDemo::onAddNewPlayer** -> PlayerRef::getPacketHandler -> SetServerCamera Packet -> Client Network

**Pipeline 2: Player Interaction (World Editing)**
> Flow:
> Client Mouse Input -> PlayerMouseButtonEvent -> **CameraDemo::onPlayerMouseButton** -> World::setBlock -> World State Change

