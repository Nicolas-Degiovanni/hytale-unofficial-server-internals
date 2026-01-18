---
description: Architectural reference for RespawnBlock
---

# RespawnBlock

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Data Component

## Definition
```java
// Signature
public class RespawnBlock implements Component<ChunkStore> {
```

## Architecture & Concepts
The RespawnBlock is a server-side Data Component within Hytale's Entity-Component-System (ECS) architecture. Its sole purpose is to associate a physical block entity in the game world with a specific player, marking it as that player's active respawn point.

This class is a simple data container, holding only the UUID of the owning player. The core logic is not implemented in this class but in the nested **RespawnBlock.OnRemove** system. This separation of data (the Component) from behavior (the System) is a fundamental principle of the engine's ECS design.

The component is managed within the scope of a **ChunkStore**, meaning its lifecycle is intrinsically tied to the world chunk it resides in. The static CODEC field indicates that this component is designed for persistence; it is serialized and deserialized as part of the chunk data, allowing respawn points to be saved and loaded with the world.

## Lifecycle & Ownership
-   **Creation:** A RespawnBlock component is instantiated and attached to a block entity when a player performs an action to set their spawn point (e.g., interacting with a bed). This operation is not performed directly but is queued via a CommandBuffer by a higher-level game logic system that processes player interactions.
-   **Scope:** The component exists for as long as the block it is attached to remains in the world and serves as the designated respawn point. It is persisted to disk along with the chunk.
-   **Destruction:** The component is removed when its associated block entity is destroyed (e.g., mined by a player). The ECS engine detects this removal and triggers the **RespawnBlock.OnRemove** system, which performs the critical cleanup logic of un-registering the respawn point from the player's data. The system explicitly ignores removals caused by chunk unloading to prevent data loss.

## Internal State & Concurrency
-   **State:** The state is minimal and mutable, consisting of a single field: ownerUUID. It is a Plain Old Java Object (POJO) intended to hold data, not logic.
-   **Thread Safety:** This component is **not thread-safe**. Like all ECS components in the engine, it is designed to be accessed and mutated exclusively by the main server thread or via thread-safe CommandBuffers. Direct modification from other threads will lead to race conditions and world corruption.

## API Surface
The public API is minimal, as interaction is primarily managed by the ECS engine and associated systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, static type identifier for this component. |
| getOwnerUUID() | UUID | O(1) | Returns the UUID of the player who owns this respawn point. |
| setOwnerUUID(UUID) | void | O(1) | Sets the UUID of the owning player. |
| clone() | Component | O(1) | Creates a shallow copy of this component. |

## Integration Patterns

### Standard Usage
Developers should never interact with this component directly. Logic for setting a respawn point should add the component to a block entity via a command buffer. The cleanup is handled automatically.

```java
// Example from a hypothetical PlayerInteractionSystem
// This code would add the component to a block entity.
UUID playerID = player.getUUID();
Ref<ChunkStore> blockEntityRef = ...; // Reference to the block entity
commandBuffer.addComponent(blockEntityRef, new RespawnBlock(playerID));
```

### Anti-Patterns (Do NOT do this)
-   **Manual Cleanup:** Do not manually find and remove the respawn point from a player's PlayerWorldData when a block is broken. The **RespawnBlock.OnRemove** system is the single source of truth for this operation. Bypassing it will cause data desynchronization, leaving players with invalid respawn points.
-   **State Management:** Do not add logic or complex state to this component. It is intended to be a simple data tag. All related logic must be implemented in an accompanying ECS System.

## Data Pipeline
The flow of data and logic is event-driven and managed by the ECS engine. The most critical pipeline is the one for destruction and cleanup.

> Flow:
> Block Destruction Event -> ECS Engine queues removal of **RespawnBlock** component -> **RespawnBlock.OnRemove** system is invoked -> System retrieves player UUID from the component -> System looks up PlayerRef via Universe -> PlayerWorldData is retrieved and modified -> Player's respawn point is officially cleared.

