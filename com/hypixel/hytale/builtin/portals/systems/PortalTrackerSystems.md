---
description: Architectural reference for PortalTrackerSystems
---

# PortalTrackerSystems

**Package:** com.hypixel.hytale.builtin.portals.systems
**Type:** Utility

## Definition
```java
// Signature
public final class PortalTrackerSystems {
```

## Architecture & Concepts
The **PortalTrackerSystems** class is a non-instantiable utility class that serves as a namespace and container for two distinct but related Entity Component Systems (ECS): **TrackerSystem** and **UiTickingSystem**.

These systems collectively manage the server-side logic for tracking player interaction with in-game portals and synchronizing portal UI state with the client. This class does not possess its own logic; it merely groups the systems responsible for:
1.  **Event-Driven Updates:** Reacting immediately when a player enters or leaves a world (**TrackerSystem**).
2.  **Periodic State Synchronization:** Regularly pushing portal status updates to all relevant players (**UiTickingSystem**).

By grouping these systems, the engine maintains a clear separation of concerns while ensuring that all portal-related tracking logic is co-located and easily discoverable.

---
description: Architectural reference for PortalTrackerSystems.TrackerSystem
---

# PortalTrackerSystems.TrackerSystem

**Package:** com.hypixel.hytale.builtin.portals.systems
**Type:** System Component

## Definition
```java
// Signature
public static class TrackerSystem extends RefSystem<EntityStore> {
```

## Architecture & Concepts
**TrackerSystem** is an event-driven system within the Hytale ECS framework. It functions as a state synchronization bridge, specifically designed to react to the presence of player entities within a world. Its core responsibility is to ensure that a player immediately receives the correct portal UI information upon joining a world and that this information is cleared upon leaving.

This system operates on a "join/leave" paradigm. It subscribes to entity addition and removal events via its **getQuery** method, which targets entities possessing both a **Player** and a **PlayerRef** component.

-   On **player addition**, it retrieves the current portal configuration from the world's **PortalWorld** resource and transmits a full state packet (**UpdatePortal**) to the newly joined player.
-   On **player removal**, it sends a "clear" packet to the departing player, effectively resetting their portal UI, and removes them from the set of players who can see the portal UI.

This design ensures that portal state is managed efficiently, avoiding unnecessary periodic checks for new players and providing immediate feedback upon world entry and exit.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS framework when a world is initialized and its associated systems are registered. It is not intended for manual creation.
-   **Scope:** The lifecycle of a **TrackerSystem** instance is tightly bound to the **World** instance it is registered with. It persists as long as the world is active.
-   **Destruction:** The instance is marked for garbage collection when its parent **World** is unloaded or the server shuts down.

## Internal State & Concurrency
-   **State:** This system is entirely stateless. It does not maintain any internal data between invocations. All necessary information is read directly from ECS components (**PlayerRef**) and world resources (**PortalWorld**) during method execution.
-   **Thread Safety:** The Hytale ECS framework guarantees that all **RefSystem** callbacks (**onEntityAdded**, **onEntityRemove**) are executed serially on the main world tick thread. Therefore, this class is inherently thread-safe within its operational context. Direct invocation from other threads is unsupported and will lead to concurrency violations.

## API Surface
The public API consists of callbacks invoked by the ECS framework, not by developers directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(...) | void | O(1) | Invoked when a player entity enters the world. Sends a full portal state packet. |
| onEntityRemove(...) | void | O(1) | Invoked when a player entity leaves the world. Sends a portal clear packet. |
| getQuery() | Query | O(1) | Defines the entity filter. The system only acts on entities with **Player** and **PlayerRef**. |

## Integration Patterns

### Standard Usage
This system is not used directly. It is registered with a world's system manager during server or world initialization. The framework then automatically invokes its methods based on entity lifecycle events.

```java
// Example of system registration (conceptual)
World world = ...;
world.getSystemRegistry().register(new PortalTrackerSystems.TrackerSystem());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PortalTrackerSystems.TrackerSystem()`. System registration is handled by engine configuration or bootstrap code.
-   **Manual Invocation:** Never call **onEntityAdded** or **onEntityRemove** directly. Doing so bypasses the ECS framework's state management and thread safety guarantees, leading to unpredictable behavior.

## Data Pipeline
The system acts as a reactive component in the player lifecycle data flow.

> Flow:
> Player Joins World -> ECS Dispatches EntityAdded Event -> **TrackerSystem.onEntityAdded** -> Reads **PortalWorld** Resource -> Writes **UpdatePortal** Packet -> Network Layer -> Player Client

---
description: Architectural reference for PortalTrackerSystems.UiTickingSystem
---

# PortalTrackerSystems.UiTickingSystem

**Package:** com.hypixel.hytale.builtin.portals.systems
**Type:** System Component

## Definition
```java
// Signature
public static class UiTickingSystem extends DelayedEntitySystem<EntityStore> {
```

## Architecture & Concepts
**UiTickingSystem** is a time-based system responsible for the periodic synchronization of portal UI state for all active players in a world. Unlike the event-driven **TrackerSystem**, this system operates on a fixed delay (1.0 second), ensuring that all players receive consistent and regular updates.

Its primary function is to handle dynamic changes in the portal's state that occur *after* a player has already joined the world. A critical use case is broadcasting the remaining time before a world instance is shut down, which is sourced from the **InstanceDataResource**.

The system's logic differentiates between two scenarios for efficiency:
1.  **New Player:** If a player is detected who was not previously known to be viewing the UI, a complete **UpdatePortal** packet is generated and sent.
2.  **Existing Player:** For players already tracked, a more lightweight "update" packet is sent, containing only the changed information.

This dual-mode approach ensures new players get the full context while minimizing network traffic for subsequent periodic updates.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS framework during world initialization, alongside other core systems.
-   **Scope:** The system's lifetime is coupled to its parent **World**. It remains active and ticking as long as the world exists.
-   **Destruction:** The instance is garbage collected when the **World** it belongs to is unloaded.

## Internal State & Concurrency
-   **State:** This system is stateless. It relies entirely on external state read from world resources (**PortalWorld**, **InstanceDataResource**) and entity components (**PlayerRef**) during each tick.
-   **Thread Safety:** As a subclass of **DelayedEntitySystem**, the ECS scheduler guarantees that the **tick** method is executed exclusively on the main world tick thread. This design prevents race conditions and ensures safe access to ECS data. Accessing this system from other threads is strictly forbidden.

## API Surface
The public contract is defined by the **tick** method, which is called by the ECS scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | O(1) | Executed once per second for each player entity. Sends a full or partial portal update packet. |
| getQuery() | Query | O(1) | Returns a query that targets entities with both **Player** and **PlayerRef** components. |

## Integration Patterns

### Standard Usage
This system is intended to be registered with a world's system manager. The framework handles its scheduling and execution automatically.

```java
// Example of system registration (conceptual)
World world = ...;
world.getSystemRegistry().register(new PortalTrackerSystems.UiTickingSystem());
```

### Anti-Patterns (Do NOT do this)
-   **Altering Tick Rate:** The 1.0-second tick rate is hardcoded. Modifying this system to tick more frequently could cause unnecessary network load and server performance degradation.
-   **Direct Invocation:** Calling the **tick** method manually is an anti-pattern. This circumvents the ECS scheduler and can lead to race conditions and inconsistent state if called from the wrong thread or at the wrong time.

## Data Pipeline
This system acts as a periodic data source, pushing state to clients on a schedule.

> Flow:
> World Tick -> ECS Scheduler -> **UiTickingSystem.tick** -> Reads **PortalWorld** & **InstanceDataResource** -> Writes **UpdatePortal** Packet -> Network Layer -> Player Client

