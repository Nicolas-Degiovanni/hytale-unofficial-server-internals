---
description: Architectural reference for CancelChainInteraction
---

# CancelChainInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class CancelChainInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The CancelChainInteraction class is a data-driven command object that represents a specific, configured game action: the termination of an active chaining interaction. It is not a service or a manager, but rather a concrete implementation of the `SimpleInstantInteraction` contract, designed to be defined within game data files (e.g., JSON or HOCON) and deserialized at runtime.

Its primary architectural role is to bridge the gap between declarative game configuration and the imperative logic of the Entity-Component System (ECS). When triggered, it translates its configured state—the `chainId`—into a direct modification of an entity's component data.

The class distinguishes between two execution phases, a pattern common in the interaction system:
1.  **Simulation (`simulateFirstRun`):** This method contains the core logic. It immediately modifies the entity's state via a `CommandBuffer`. This provides instant feedback within the game simulation, ensuring that subsequent logic in the same tick sees the chain as cancelled.
2.  **Authoritative Run (`firstRun`):** This method is intentionally empty. It signifies that the primary effect of this interaction is simulated and does not require a separate, delayed server-side process.

Finally, it is responsible for generating the corresponding network packet (`com.hypixel.hytale.protocol.CancelChainInteraction`) to inform the client that the interaction chain has been broken.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the `BuilderCodec` system during the server's asset loading phase. Game designers define the properties of this interaction in configuration files, and the static `CODEC` field handles the deserialization and instantiation. Direct construction via `new` is an invalid use case.
-   **Scope:** An instance of CancelChainInteraction is effectively immutable after creation. It persists in memory as part of a larger, static interaction graph for as long as the server's game definitions are loaded. It is a stateless template that is referenced, but not cloned, when an interaction is triggered.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup when the server unloads the asset packs or configurations that define it. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The class holds a single piece of immutable state: the `chainId` string. This value is injected once during deserialization by the `CODEC` and is never modified during the object's lifetime.
-   **Thread Safety:** The object itself is inherently thread-safe due to its immutable state. However, its methods are designed to operate on mutable, thread-bound context objects like `InteractionContext` and `CommandBuffer`.

    **Warning:** Invoking methods such as `simulateFirstRun` from any thread other than the entity's primary update thread will result in race conditions and data corruption. The interaction system is responsible for ensuring all method invocations occur within the correct, synchronized game tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generatePacket() | Interaction | O(1) | Creates a new, unconfigured network packet to notify clients of the cancellation. |
| configurePacket(Interaction) | void | O(1) | Populates the provided network packet with the configured `chainId`. |
| firstRun(...) | void | O(1) | No-op. The interaction's logic is fully handled during the simulation phase. |
| simulateFirstRun(...) | void | O(1) | The core logic. Posts a command to the `CommandBuffer` to remove the `chainId` from the target entity's `ChainingInteraction.Data` component. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is automatically managed and executed by the server's core interaction module. The system identifies the active interaction from game data and invokes its methods against the current `InteractionContext`.

A conceptual view of the system's usage:
```java
// Pseudo-code representing the interaction system's internal logic

// 1. An interaction is triggered on an entity
InteractionDefinition definition = getTriggeredInteraction(); // e.g., "player.cancel_spell"
CancelChainInteraction action = definition.getActionOfType(CancelChainInteraction.class);

// 2. The system provides the context and invokes the method
InteractionContext context = createInteractionContextForEntity(entity);
CooldownHandler cooldowns = getCooldownsForEntity(entity);

if (action != null) {
    // This is the primary path of execution
    action.simulateFirstRun(InteractionType.PRIMARY, context, cooldowns);

    // The system then creates and sends the packet
    Interaction packet = action.generatePacket();
    action.configurePacket(packet);
    entity.sendPacket(packet);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new CancelChainInteraction()`. This bypasses the `CODEC` system, leaving the critical `chainId` field as null. Subsequent method calls will fail with a `NullPointerException`.
-   **State Mutation:** Do not use reflection or other means to modify the `chainId` after the object has been constructed. This violates the configuration-as-code contract and will lead to unpredictable behavior.
-   **External Invocation:** Do not call `simulateFirstRun` outside of the managed interaction lifecycle. The `InteractionContext` and `CommandBuffer` it receives are highly stateful and only valid for a single operation within a specific game tick.

## Data Pipeline
The data flow for this component begins with game configuration and ends with a component state change and a network packet.

> Flow:
> Game Config File (JSON/HOCON) -> Server Asset Loader -> **`CODEC` Deserializer** -> In-memory `CancelChainInteraction` instance -> Game Event Trigger -> `simulateFirstRun` execution -> `CommandBuffer` command -> Entity Component State Change
>
> Parallel Flow:
> Game Event Trigger -> `generatePacket` / `configurePacket` -> Outbound Network Packet -> Client Receives Cancellation


