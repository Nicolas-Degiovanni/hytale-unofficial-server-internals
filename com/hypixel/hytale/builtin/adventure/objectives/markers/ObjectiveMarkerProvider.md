---
description: Architectural reference for ObjectiveMarkerProvider
---

# ObjectiveMarkerProvider

**Package:** com.hypixel.hytale.builtin.adventure.objectives.markers
**Type:** Singleton

## Definition
```java
// Signature
public class ObjectiveMarkerProvider implements WorldMapManager.MarkerProvider {
```

## Architecture & Concepts
The **ObjectiveMarkerProvider** serves as a specialized data bridge, translating state from the Adventure Objective system into visual markers for the World Map system. It implements the **WorldMapManager.MarkerProvider** interface, establishing a clear contract that allows the **WorldMapManager** to discover and query for map markers without needing any direct knowledge of objectives, quests, or tasks.

This class operates as a passive, stateless adapter. It is invoked by the **WorldMapManager** during its update cycle for a specific player. Its sole responsibility is to inspect the player's currently active objectives, extract any associated **MapMarker** data from the active tasks, and submit them to the player's **WorldMapTracker**. This decouples the high-level game logic of objectives from the low-level rendering and networking logic of the world map.

### Lifecycle & Ownership
- **Creation:** The **ObjectiveMarkerProvider** is a static singleton, exposed via the public final field INSTANCE. It is instantiated by the Java ClassLoader when the class is first referenced, typically during server bootstrap when the **WorldMapManager** is initialized.
- **Scope:** Application-wide. A single instance exists for the entire server lifetime.
- **Destruction:** The instance is reclaimed by the garbage collector only when the server process shuts down. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** This provider is entirely stateless. It contains no instance fields and does not cache any data between invocations of its **update** method. All necessary context, such as the player and world, is provided as arguments to the method. This design ensures that every update is idempotent and based on the most current game state.

- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The **update** method is a pure function with respect to the provider's own state. However, it reads from shared data structures like **ObjectiveDataStore** and writes to the **WorldMapTracker**. The calling system, the **WorldMapManager**, is responsible for ensuring that calls to **update** are made from a thread that can safely access these systems, typically the main world thread.

## API Surface
The public contract is defined by the **WorldMapManager.MarkerProvider** interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| update(world, config, tracker, ...) | void | O(N) | Called by the **WorldMapManager**. Iterates through a player's active objectives and their tasks to find associated map markers. N is the total number of markers across all active tasks for the player. |

## Integration Patterns
This component is not designed for direct invocation by general game logic. It is a system-level provider intended for registration with a manager.

### Standard Usage
The standard and only correct usage of this class is to register its singleton INSTANCE with the **WorldMapManager** during server initialization. The manager will then orchestrate calls to the **update** method at the appropriate time for each player.

*Conceptual Registration (Actual code may differ):*
```java
// During server or world initialization
WorldMapManager mapManager = world.getWorldMapManager();
mapManager.registerProvider(ObjectiveMarkerProvider.INSTANCE);
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call `ObjectiveMarkerProvider.INSTANCE.update(...)` directly. This bypasses the orchestration and caching layers of the **WorldMapManager**, potentially leading to redundant network packets, incorrect marker visibility, or race conditions.
- **Instantiation:** Do not attempt to create a new instance of this class. The constructor is private to enforce the singleton pattern. Always use the static **INSTANCE** field.

## Data Pipeline
The **ObjectiveMarkerProvider** sits in the middle of a data flow that transforms abstract game state into concrete network data for the client.

> Flow:
> Player State -> PlayerConfigData (Active Objective UUIDs) -> ObjectiveDataStore -> **ObjectiveMarkerProvider** -> WorldMapTracker -> Network Packet (to Client)

