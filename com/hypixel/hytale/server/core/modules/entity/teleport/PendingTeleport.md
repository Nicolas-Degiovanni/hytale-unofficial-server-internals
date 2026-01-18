---
description: Architectural reference for PendingTeleport
---

# PendingTeleport

**Package:** com.hypixel.hytale.server.core.modules.entity.teleport
**Type:** Transient Component

## Definition
```java
// Signature
public class PendingTeleport implements Component<EntityStore> {
```

## Architecture & Concepts
The PendingTeleport component is a server-side, stateful data container within the Entity-Component-System (ECS) architecture. Its primary function is to manage and validate the client-server handshake during an entity's teleportation sequence.

This component acts as a transactional queue for teleport requests for a single entity. When the server commands an entity to teleport, it does not immediately update the entity's authoritative position. Instead, it enqueues a teleport request within this component and sends a corresponding packet to the client. The server then awaits a confirmation packet from the client.

The PendingTeleport component is the mechanism that bridges the server's command and the client's acknowledgement. It validates the client's response against the expected teleport ID and destination, preventing desynchronization and validating that the client has correctly processed the teleport instruction. This is a critical pattern for maintaining world consistency in a high-latency environment.

### Lifecycle & Ownership
- **Creation:** An instance of PendingTeleport is created and attached to an entity by a server-side system (e.g., a TeleportSystem) the first time a teleport is initiated for that entity. The static `getComponentType` method indicates it is managed and registered via the central EntityModule.
- **Scope:** The component's lifetime is tied to an active teleportation sequence. It persists on the entity as long as there are teleports in its queue. Once the queue is empty and the final teleport is validated, the component is typically removed from the entity to conserve memory.
- **Destruction:** The component is marked for garbage collection when it is detached from its parent entity by a managing system.

## Internal State & Concurrency
- **State:** This component is highly mutable. It maintains an internal queue of pending Teleport objects, the last confirmed position, and two ID counters (`nextTeleportId` and `lastTeleportId`) to track the sequence. This state is fundamental to its validation logic.
- **Thread Safety:** **This component is not thread-safe.** It is designed to be exclusively accessed and modified by the server's main game loop thread, which processes entity updates. Any concurrent access from other threads (e.g., network I/O threads) without external synchronization will lead to race conditions and severe state corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | Retrieves the registered type definition for this component from the EntityModule. |
| validate(teleportId, teleportPosition) | Result | O(1) | Core validation logic. Confirms a client's response matches the expected ID and position. |
| isEmpty() | boolean | O(1) | Checks if the teleport queue is empty. |
| queueTeleport(teleport) | int | O(1) | Adds a new teleport request to the queue and returns its unique transaction ID. |
| getPosition() | Vector3d | O(1) | Returns the last successfully validated teleport position. |

## Integration Patterns

### Standard Usage
The component is intended to be managed by server-side systems. A system initiates a teleport, and a network handler validates the client's response.

```java
// In a server system that initiates a teleport
Entity entity = ...;
PendingTeleport pt = entity.getOrCreateComponent(PendingTeleport.getComponentType());
int teleportId = pt.queueTeleport(new Teleport(destination));
server.getNetworkManager().sendTeleportPacket(entity.getOwner(), teleportId, destination);

// In a network packet handler for client teleport confirmation
Entity entity = ...;
PendingTeleport pt = entity.getComponent(PendingTeleport.getComponentType());
if (pt != null) {
    PendingTeleport.Result result = pt.validate(packet.getTeleportId(), packet.getPosition());
    // Handle OK, INVALID_ID, or INVALID_POSITION results
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PendingTeleport()`. The ECS framework must manage component creation via `entity.getOrCreateComponent`. Direct instantiation bypasses the entity's component registry.
- **State Tampering:** Do not manually access or modify the internal `pendingTeleports` list or the ID counters. Use the provided `queueTeleport` and `validate` methods to ensure transactional integrity.
- **Ignoring Validation Result:** The `Result` enum from the `validate` method is critical. Failing to handle `INVALID_ID` or `INVALID_POSITION` can lead to client-server desynchronization or allow for position exploits.

## Data Pipeline
The flow of data and control for a single teleport operation is a round-trip from the server to the client and back.

> Flow:
> Server-Side System -> `queueTeleport` -> **PendingTeleport** (State Update) -> Outbound Network Packet -> Client Rendering -> Inbound Network Packet -> Server Network Handler -> `validate` -> **PendingTeleport** (State Update) -> Authoritative Entity Position Update

