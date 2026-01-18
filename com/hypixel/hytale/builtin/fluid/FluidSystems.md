---
description: Architectural reference for FluidSystems
---

# FluidSystems

**Package:** com.hypixel.hytale.builtin.fluid
**Type:** Utility

## Definition
```java
// Signature
public class FluidSystems {
```

## Architecture & Concepts
FluidSystems is a static container class that serves as a namespace for a collection of related Entity Component Systems (ECS) responsible for the entire lifecycle of fluids in the game world. It is not instantiated and holds no state. Its purpose is to logically group the systems that handle fluid creation, data migration, simulation, and network replication.

The nested classes within FluidSystems define a clear, ordered pipeline for fluid management on the server:
1.  **Initialization & Migration:** When a chunk is loaded, **EnsureFluidSection** and **MigrateFromColumn** guarantee that a valid **FluidSection** component exists and that data from legacy world formats is correctly migrated. **SetupSection** then performs final initialization.
2.  **Simulation:** The **Ticking** system executes the core fluid dynamics logic (e.g., flow, spread) for each chunk section on every server tick.
3.  **Replication:** The **ReplicateChanges** system detects modifications to fluid data and broadcasts them to clients. The **LoadPacketGenerator** handles sending the full fluid state to players when they first load a chunk.

This modular design isolates different concerns, allowing the engine to manage dependencies and execution order precisely.

---
description: Architectural reference for FluidSystems.EnsureFluidSection
---

# FluidSystems.EnsureFluidSection

**Package:** com.hypixel.hytale.builtin.fluid
**Type:** ECS System (HolderSystem)

## Definition
```java
// Signature
public static class EnsureFluidSection extends HolderSystem<ChunkStore> {
```

## Architecture & Concepts
This system acts as a data integrity guard. Its sole responsibility is to ensure that any entity representing a game world chunk section (**ChunkSection**) also has a corresponding **FluidSection** component.

It operates on a simple principle: if an entity has a **ChunkSection** but lacks a **FluidSection**, this system will immediately add a new, empty **FluidSection**. This prevents downstream systems, such as **Ticking** or **ReplicateChanges**, from failing due to missing components and simplifies their logic, as they can always assume a **FluidSection** is present. It is a foundational setup system that runs at the very beginning of the chunk lifecycle.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS framework during system registration at startup.
-   **Scope:** Singleton within the ECS world. It persists for the entire server session.
-   **Destruction:** Decommissioned when the server shuts down and the ECS world is destroyed.

## Internal State & Concurrency
-   **State:** Stateless. The system contains no mutable fields and operates only on the components of the entity it is processing.
-   **Thread Safety:** Fully thread-safe. As a stateless system reacting to entity creation events, it can be safely executed by any worker thread in the ECS framework.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Event handler triggered when an entity matching the query is created. Adds a new **FluidSection** component. |

## Integration Patterns

### Standard Usage
This system is automatically registered and managed by the engine's ECS framework. No direct developer interaction is required. It is triggered implicitly when new chunk sections are loaded into the world.

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Do not call **onEntityAdd** directly. The ECS framework is responsible for its invocation based on entity lifecycle events.
-   **Component Pre-addition:** Do not manually add a **FluidSection** before this system runs. While not harmful, it is redundant and subverts the system's purpose as a data integrity guarantee.

## Data Pipeline
> Flow:
> Chunk Load -> Entity with **ChunkSection** created -> **EnsureFluidSection** query matches -> **onEntityAdd** triggered -> New **FluidSection** component added to entity

---
description: Architectural reference for FluidSystems.LoadPacketGenerator
---

# FluidSystems.LoadPacketGenerator

**Package:** com.hypixel.hytale.builtin.fluid
**Type:** ECS System (LoadFuturePacketDataQuerySystem)

## Definition
```java
// Signature
public static class LoadPacketGenerator extends ChunkStore.LoadFuturePacketDataQuerySystem {
```

## Architecture & Concepts
This system is a critical component of the server's world streaming architecture. When a player's client needs to load a new chunk, the server queries for systems that can provide the necessary data packets. **LoadPacketGenerator** responds to this query by generating packets that describe the complete fluid state for all sections within a chunk column.

It retrieves a pre-compressed, cached packet from each **FluidSection** component. This is a significant performance optimization, as it avoids re-serializing and re-compressing the same fluid data for every player who loads the chunk. The result is provided as a **CompletableFuture**, allowing the network pipeline to process packet generation asynchronously without blocking the main server thread.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS framework during system registration.
-   **Scope:** Singleton within the ECS world. Persists for the entire server session.
-   **Destruction:** Decommissioned upon server shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** The **fetch** method is designed to be called from multiple threads. It interacts with **CompletableFuture** and thread-safe component data structures, ensuring safe concurrent execution during the world streaming process. The exception handling within the future chain prevents a single failed packet from halting the entire chunk data transmission.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fetch(...) | void | O(N) | Populates a list with **CompletableFuture** objects, where N is the number of sections in the chunk. Each future resolves to a network **Packet** containing fluid data. |

## Integration Patterns

### Standard Usage
This system is invoked automatically by the server's chunk loading and player synchronization logic. A developer working on world generation or streaming would not interact with this class directly.

```java
// Conceptual engine-level usage
List<CompletableFuture<Packet>> packets = new ArrayList<>();
PlayerRef player = ...;
ArchetypeChunk<ChunkStore> chunk = ...;

// The engine finds all LoadFuturePacketDataQuerySystem instances, including this one
// and invokes fetch() on them.
loadPacketGenerator.fetch(index, chunk, store, cmdBuffer, player, packets);

// The engine then waits for all futures and sends the resulting packets.
```

## Data Pipeline
> Flow:
> Player moves into new area -> Server initiates chunk load for player -> **LoadPacketGenerator.fetch** is invoked -> Retrieves cached packet from **FluidSection** -> Adds **CompletableFuture<Packet>** to results list -> Server network layer sends resolved packet to client

---
description: Architectural reference for FluidSystems.MigrateFromColumn
---

# FluidSystems.MigrateFromColumn

**Package:** com.hypixel.hytale.builtin.fluid
**Type:** ECS System (ChunkColumnMigrationSystem)

## Definition
```java
// Signature
public static class MigrateFromColumn extends ChunkColumnMigrationSystem {
```

## Architecture & Concepts
This system is a specialized data migration utility. Its purpose is to support backward compatibility by upgrading fluid data from a legacy storage format to the current component-based architecture.

When a chunk is loaded from disk, this system checks for deprecated fluid data stored within the old **BlockSection** format. If found, it extracts this data using **takeMigratedFluid** and moves it into a modern **FluidSection** component on the corresponding chunk section entity. This operation is idempotent and transactional; once the data is moved, the source is cleared, ensuring the migration only happens once per chunk section.

The system's dependency on running *before* **LegacyModule.MigrateLegacySections** is critical, establishing a strict order of operations for world data upgrades.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS framework at startup.
-   **Scope:** Singleton within the ECS world.
-   **Destruction:** Decommissioned upon server shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Thread-safe. The system operates on a single entity at a time during the chunk loading process, which is itself a thread-safe operation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(N) | Event handler triggered when a chunk column entity is created. Iterates through N sections, migrating legacy fluid data if present. |

## Integration Patterns

### Standard Usage
This system is fully automated. It is triggered by the ECS framework during the loading of any chunk that might contain legacy data. No developer interaction is needed.

### Anti-Patterns (Do NOT do this)
-   **Altering Dependencies:** Changing the execution order of this system relative to other migration systems can lead to data corruption or incomplete world upgrades. Its dependencies are carefully configured and must not be altered.

## Data Pipeline
> Flow:
> Legacy Chunk loaded from disk -> Entity created with **BlockChunk** component containing old data -> **MigrateFromColumn.onEntityAdd** triggered -> Extracts fluid data from **BlockSection** -> Puts data into **FluidSection** component -> Marks chunk as needing to be saved

---
description: Architectural reference for FluidSystems.ReplicateChanges
---

# FluidSystems.ReplicateChanges

**Package:** com.hypixel.hytale.builtin.fluid
**Type:** ECS System (EntityTickingSystem)

## Definition
```java
// Signature
public static class ReplicateChanges extends EntityTickingSystem<ChunkStore> implements RunWhenPausedSystem<ChunkStore> {
```

## Architecture & Concepts
This system is the primary mechanism for synchronizing fluid state changes from the server to connected clients. It runs every tick, polling each **FluidSection** for modifications.

The core logic uses a highly efficient, two-stage replication strategy:
1.  **Delta Compression:** For a small number of changes (less than 1024), it sends discrete update packets (**ServerSetFluid** for a single change, **ServerSetFluids** for a batch). This minimizes network bandwidth for minor fluid movements.
2.  **Full Compression:** If the number of changes is large (e.g., a large body of water flowing), sending individual updates is inefficient. In this case, the system instead sends the complete, pre-cached, compressed packet for the entire chunk section. This is the same packet generated by **LoadPacketGenerator**.

This system is also responsible for filtering updates, sending packets only to players who have the relevant chunk loaded, which is determined by querying the player's **ChunkTracker**.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS framework.
-   **Scope:** Singleton within the ECS world.
-   **Destruction:** Decommissioned upon server shutdown.

## Internal State & Concurrency
-   **State:** Stateless. It reads and clears the change set from the **FluidSection** component within its tick method.
-   **Thread Safety:** This system is designed for parallel execution. The **tick** method can be called concurrently for different chunk sections. It safely interacts with shared world state by creating a defensive copy of the player list (**ObjectArrayList**) for use in asynchronous packet sending callbacks. It also dispatches lighting updates to the main world thread via **world.execute** to avoid race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, archetype, store, cmd) | void | O(C + P) | Runs every tick. C is the number of fluid changes, P is the number of players. Polls for changes and sends appropriate network packets. |

## Integration Patterns

### Standard Usage
This system is automatically ticked by the ECS framework. Any system that modifies fluid state (e.g., **FluidSystems.Ticking**, player actions) will have its changes automatically detected and replicated by this system.

### Anti-Patterns (Do NOT do this)
-   **Bypassing Change Tracking:** Modifying a **FluidSection** without correctly marking the changed positions will render this system ineffective. The changes will occur on the server but will never be sent to clients, causing a state desynchronization. Always use the provided setter methods on **FluidSection**.

## Data Pipeline
> Flow:
> Fluid Simulation modifies **FluidSection** -> **ReplicateChanges.tick** runs -> Calls **getAndClearChangedPositions** -> If changes exist, determines optimal packet type -> Iterates players with chunk loaded -> Sends **ServerSetFluid(s)** or full chunk packet

---
description: Architectural reference for FluidSystems.SetupSection
---

# FluidSystems.SetupSection

**Package:** com.hypixel.hytale.builtin.fluid
**Type:** ECS System (HolderSystem)

## Definition
```java
// Signature
public static class SetupSection extends HolderSystem<ChunkStore> {
```

## Architecture & Concepts
This is a final initialization system in the chunk loading pipeline. After a **FluidSection** component has been guaranteed to exist (by **EnsureFluidSection**) and populated with any legacy data (by **MigrateFromColumn**), this system performs the final setup step.

It calls the **load** method on the **FluidSection** component, passing in the section's world coordinates (X, Y, Z). This call allows the component to initialize any internal data structures that are dependent on its absolute position in the world. Its dependency on running *after* **MigrateFromColumn** is critical to ensure it operates on fully migrated and correct data.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS framework at startup.
-   **Scope:** Singleton within the ECS world.
-   **Destruction:** Decommissioned upon server shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Thread-safe. It is triggered by entity creation events during the chunk loading process and operates on a single entity's components at a time.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, reason, store) | void | O(1) | Event handler triggered when an entity with a **FluidSection** is added. Calls the **load** method on the component. |

## Integration Patterns

### Standard Usage
This system is fully automated and managed by the ECS framework. It ensures that all **FluidSection** components are correctly initialized before they are used by simulation or replication systems.

## Data Pipeline
> Flow:
> Entity with **FluidSection** created -> **SetupSection.onEntityAdd** triggered -> Calls **fluidSectionComponent.load(x, y, z)** -> Component is now fully initialized

---
description: Architectural reference for FluidSystems.Ticking
---

# FluidSystems.Ticking

**Package:** com.hypixel.hytale.builtin.fluid
**Type:** ECS System (EntityTickingSystem)

## Definition
```java
// Signature
public static class Ticking extends EntityTickingSystem<ChunkStore> {
```

## Architecture & Concepts
This system is the heart of the server-side fluid simulation. It is responsible for executing the dynamic behavior of fluids, such as flowing, spreading, and settling.

On each server tick, this system iterates through every chunk section that contains a **FluidSection**. For each section, it checks for blocks that are scheduled to be ticked. For each such block, it invokes the **tick** method on the corresponding **Fluid.FluidTicker** asset. This design is data-driven; the behavior of a fluid is defined in its asset data, not hardcoded in this system.

To optimize performance, it uses a **FluidTicker.CachedAccessor**. This object pre-fetches and caches references to neighboring chunk sections, dramatically reducing the cost of repeated lookups, which are common in block-based simulation algorithms.

The system's dependencies place it precisely within the main server tick loop, ensuring fluid updates happen in a deterministic order relative to other block updates.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's ECS framework.
-   **Scope:** Singleton within the ECS world.
-   **Destruction:** Decommissioned upon server shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** This system is designed for parallel execution across different chunk sections. The use of a **CommandBuffer** and the **CachedAccessor** ensures that modifications to world state are handled in a thread-safe manner, with changes being queued up for later application.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, archetype, store, cmd) | void | O(N) | Runs every tick. N is the number of blocks scheduled for ticking in the section. Executes fluid simulation logic for each ticking block. |

## Integration Patterns

### Standard Usage
This system is automatically ticked by the ECS framework. To create a new fluid, a developer would define a new **Fluid** asset and implement the **FluidTicker** interface. The engine will automatically discover and execute this new logic via the **Ticking** system.

### Anti-Patterns (Do NOT do this)
-   **Direct Modification:** Modifying fluid state from other systems during the ticking phase can lead to race conditions and unpredictable simulation behavior. All fluid simulation logic should be encapsulated within a **FluidTicker**.
-   **Ignoring the Accessor:** Writing a **FluidTicker** that directly queries the world for neighboring chunks instead of using the provided **CachedAccessor** will result in severe performance degradation.

## Data Pipeline
> Flow:
> Server Tick begins -> **Ticking.tick** is called for a chunk section -> Iterates ticking blocks -> For each block, gets **Fluid** asset -> Calls **fluid.getTicker().tick()** -> Ticker logic modifies **FluidSection** via **CommandBuffer** -> Changes are later replicated by **ReplicateChanges** system

