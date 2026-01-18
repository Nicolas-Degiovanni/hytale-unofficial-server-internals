---
description: Architectural reference for InteractionPriority
---

# InteractionPriority

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Value Object / Data Carrier

## Definition
```java
// Signature
public record InteractionPriority(@Nullable Map<PrioritySlot, Integer> values) implements NetworkSerializable<com.hypixel.hytale.protocol.InteractionPriority> {
```

## Architecture & Concepts
InteractionPriority is an immutable data carrier that defines the precedence of a game interaction. It is a fundamental configuration component within the server's interaction resolution system. When a player attempts to interact with the world, multiple potential actions might be valid (e.g., using an item in the main hand vs. interacting with a block). This class provides the mechanism to deterministically select the correct one.

Its primary role is to hold a map of *slots* to *priority values*. The system queries this object for a given PrioritySlot (e.g., MainHand, OffHand) to retrieve a numerical priority. Higher values indicate higher precedence.

A key architectural feature is its implementation of the NetworkSerializable interface. This signifies that an InteractionPriority configuration, defined on the server, can be serialized into a network packet and synchronized with the client. This is essential for ensuring that client-side predictive systems and UI elements can accurately reflect server-side interaction logic.

The class is implemented as a Java record, enforcing immutability and making it an ideal candidate for a simple, thread-safe configuration value.

## Lifecycle & Ownership
- **Creation:** InteractionPriority instances are not managed services. They are typically created in one of two ways:
    1. Deserialized from asset configuration files (e.g., a block or item JSON definition) by the engine's asset loading system using the provided static CODEC.
    2. Instantiated programmatically by game logic when defining dynamic or procedural content.
- **Scope:** The lifetime of an InteractionPriority object is tied to its owning configuration object, such as an ItemDefinition or a BlockState. It is a transient object, not a session-scoped singleton.
- **Destruction:** Instances are managed by the Java garbage collector. They are eligible for collection as soon as the game asset they are part of is unloaded. No manual cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. As a Java record, its state (the `values` map) is set at construction and cannot be modified thereafter. The internal map is nullable to represent a default or zero-priority state efficiently.
- **Thread Safety:** **Fully thread-safe**. Due to its immutability, an instance of InteractionPriority can be safely accessed and read by multiple threads simultaneously without any need for external synchronization or locks. This is critical in a multi-threaded server environment where game logic and asset loading may occur on different threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| InteractionPriority(int) | constructor | O(1) | Convenience constructor to create a priority with a single default value. |
| getPriority(PrioritySlot) | int | O(1) | Retrieves the priority for a slot. Implements a fallback: checks the specific slot, then the Default slot, then returns 0. |
| toPacket() | InteractionPriority | O(N) | Converts the object into its network packet representation. N is the number of entries in the map. |

## Integration Patterns

### Standard Usage
InteractionPriority should be treated as a read-only configuration value. It is typically retrieved from a game asset and used by the interaction system to compare and resolve competing actions.

```java
// In a hypothetical interaction resolution system
void resolveInteraction(Player player, Block targetBlock) {
    // Get priority from the item the player is holding
    Item heldItem = player.getHeldItem();
    InteractionPriority itemPriority = heldItem.getDefinition().getInteractionPriority();
    int itemValue = itemPriority.getPriority(PrioritySlot.MainHand);

    // Get priority from the block being targeted
    InteractionPriority blockPriority = targetBlock.getDefinition().getInteractionPriority();
    int blockValue = blockPriority.getPriority(PrioritySlot.Default);

    if (itemValue > blockValue) {
        // Execute the item's interaction
    } else {
        // Execute the block's interaction
    }
}
```

### Anti-Patterns (Do NOT do this)
- **External Null Checks:** Do not check if the internal `values` map is null before calling getPriority. The method's internal logic is designed to handle the null case gracefully by returning a default value of 0. Rely on the public API contract.
- **Modifying Input Maps:** Although records provide shallow immutability, passing a mutable map to the constructor and modifying it later violates the design contract and can lead to unpredictable behavior in a multi-threaded context. Always treat the input map as consumed and owned by the new record instance.

## Data Pipeline
The primary flow for this class is during asset loading and network synchronization, not real-time gameplay processing.

> **Configuration Loading Flow:**
> Asset File (e.g., item.json) -> Hytale Codec Framework -> **InteractionPriority.CODEC** -> **InteractionPriority Instance** -> Attached to Game Asset Definition

> **Network Synchronization Flow:**
> Server-Side Game Asset -> **InteractionPriority Instance** -> toPacket() -> Network Packet -> Protocol Layer -> Client Machine

