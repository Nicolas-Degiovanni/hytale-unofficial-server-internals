---
description: Architectural reference for ChangeActiveSlotInteraction
---

# ChangeActiveSlotInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Configuration Object

## Definition
```java
// Signature
public class ChangeActiveSlotInteraction extends Interaction {
```

## Architecture & Concepts
The ChangeActiveSlotInteraction is a concrete implementation of the abstract **Interaction** class. It represents a single, atomic server-side operation: changing a living entity's active hotbar slot. This class is a fundamental component of Hytale's data-driven interaction system, where game mechanics are defined as configurable data rather than hard-coded logic.

Architecturally, this class acts as a stateless "verb" or "command". It does not hold the state of the entity it operates on; instead, it receives all necessary context—such as the target entity and the command buffer—via an **InteractionContext** object during execution. Its primary role is to apply a specific state change to the game world based on its pre-configured properties.

A key feature of this interaction is its ability to fork a subsequent interaction. After changing the slot, it can trigger a new **SwapTo** interaction, allowing for chained behaviors and more complex, emergent game mechanics. This makes it a building block for more sophisticated inventory management systems.

## Lifecycle & Ownership
-   **Creation:** Instances are not created programmatically by developers. They are instantiated by the engine's **Codec** system when parsing game configuration files (e.g., JSON definitions for items or abilities). The static **CODEC** field defines how to serialize and deserialize this object.
-   **Scope:** An instance of ChangeActiveSlotInteraction is a lightweight, reusable configuration object. A single instance may be held by the **InteractionManager** and reused for every execution of this action. Its lifetime is tied to the lifetime of the loaded game assets.
-   **Destruction:** Instances are managed by the Java Garbage Collector. There is no manual cleanup required. They are garbage collected when the associated game configuration is unloaded.

## Internal State & Concurrency
-   **State:** The class holds minimal internal state, primarily the **targetSlot** which is loaded from configuration. This state is considered immutable after the initial decoding process. All dynamic, per-execution state is passed in via the **InteractionContext**.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be executed exclusively on the main server thread as part of the game loop. It performs direct mutations on game state objects like **LivingEntity** and writes to the **CommandBuffer**. Accessing or executing this class from any other thread will lead to state corruption, race conditions, and server instability.

## API Surface
The primary contract is the protected **tick0** method, which is invoked by the server's **InteractionManager**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | protected void | O(1) | Executes the core logic of changing the hotbar slot. Mutates entity state via the provided InteractionContext. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns **None**, indicating this is an immediate, server-authoritative action that does not wait for client input. |
| generatePacket() | Interaction | O(1) | Creates the network packet used to synchronize the state change to the client. |
| configurePacket(packet) | protected void | O(1) | Populates the network packet with instance-specific data, such as the targetSlot. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code. Instead, it is defined within a data file that configures a broader game mechanic. The engine's **InteractionManager** is responsible for resolving and executing it.

A developer would typically define this interaction in a configuration file similar to the conceptual example below:

```json
// Example: Part of an item's ability definition
{
  "id": "*Default_Swap",
  "cooldown": {
    "id": "ChangeActiveSlot",
    "duration": 0.0
  },
  "interaction": {
    "type": "ChangeActiveSlotInteraction"
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ChangeActiveSlotInteraction()`. The object is incomplete without being configured and managed by the engine's asset loading and interaction systems.
-   **Manual Execution:** Never call the **tick0** method directly. This bypasses critical lifecycle management, state validation, and cooldown handling performed by the **InteractionManager**.
-   **Multi-threaded Access:** Do not share instances of this class or its context across threads. All interaction logic must be confined to the main server thread.

## Data Pipeline
The flow of data and control for this interaction is managed entirely by the server's core interaction loop.

> Flow:
> Player Input or Game Event -> **InteractionManager** resolves the interaction -> A new **InteractionContext** is created -> **ChangeActiveSlotInteraction.tick0()** is called -> **LivingEntity** inventory state is mutated -> A **ChangeActiveSlotInteraction** network packet is generated and sent to the client.

