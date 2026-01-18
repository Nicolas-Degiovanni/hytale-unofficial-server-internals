---
description: Architectural reference for FilterType
---

# FilterType

**Package:** com.hypixel.hytale.server.core.inventory.container.filter
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum FilterType {
```

## Architecture & Concepts
The FilterType enumeration defines a strict, finite set of policies governing item movement within inventory slots. It serves as a foundational component in the inventory system, replacing ambiguous boolean flags with a type-safe, self-documenting contract.

This component's primary architectural role is to enforce container behavior rules consistently across both the server and client. By encapsulating input and output permissions into a single state, it simplifies the logic required to validate player actions such as dragging items, shift-clicking, or automated transfers via hoppers.

The inclusion of a public static **CODEC** field is a critical design choice. It signals that FilterType is not merely an internal logic primitive but a core part of the game's data model. This allows container definitions, including their slot-specific filter policies, to be reliably serialized for network transmission or persistence.

## Lifecycle & Ownership
- **Creation:** All instances of FilterType are created and initialized by the Java Virtual Machine (JVM) during class loading. Application code does not and cannot instantiate this enum.
- **Scope:** As a static enumeration, all instances (ALLOW_INPUT_ONLY, ALLOW_OUTPUT_ONLY, etc.) are globally accessible and persist for the entire lifetime of the server process.
- **Destruction:** The instances are garbage collected only when the JVM shuts down. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** FilterType is **deeply immutable**. Its internal state, the *input* and *output* booleans, is set once via its private constructor during JVM initialization and can never be modified thereafter.
- **Thread Safety:** This enumeration is inherently **thread-safe**. Its instances can be safely accessed, passed, and evaluated by any number of concurrent threads without requiring locks or any other synchronization mechanisms. This is a direct result of its immutability.

## API Surface
The public contract is minimal and focused on querying the defined policy.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | Codec<FilterType> | N/A | A static serializer/deserializer for network and disk I/O. |
| allowInput() | boolean | O(1) | Returns true if the policy permits items to be placed into a slot. |
| allowOutput() | boolean | O(1) | Returns true if the policy permits items to be taken from a slot. |

## Integration Patterns

### Standard Usage
FilterType is intended to be used by container and inventory slot implementations to guard transfer operations. Logic should retrieve the filter from a slot definition and query its permissions before modifying the inventory state.

```java
// Example: Validating an item transfer into a slot
public boolean canInsertItem(int slotIndex, ItemStack stack) {
    InventorySlot slot = this.getSlot(slotIndex);
    FilterType filter = slot.getFilterType(); // Retrieve the policy

    if (!filter.allowInput()) {
        return false; // Operation denied by policy
    }

    // ... additional validation logic ...
    return true;
}
```

### Anti-Patterns (Do NOT do this)
- **Boolean Flags:** Avoid implementing new systems that use separate boolean flags like *canAcceptItems* and *canProvideItems*. This pattern is error-prone and has been deprecated in favor of the unified FilterType enumeration.
- **String Comparison:** Never use the enum's string representation for logic checks. This is inefficient and brittle. Always use direct object comparison.
    - **BAD:** `if (slot.getFilterType().name().equals("ALLOW_ALL"))`
    - **GOOD:** `if (slot.getFilterType() == FilterType.ALLOW_ALL)`
- **Remote Instantiation:** Attempting to instantiate this enum via reflection or other mechanisms will break the JVM's guarantees and lead to undefined behavior.

## Data Pipeline
FilterType is a data component that defines behavior. It flows from server-side configuration, through the network, to the client to ensure a consistent user experience and prevent invalid client-side actions.

> Flow:
> Server-Side Container Definition -> Serialization (using **FilterType.CODEC**) -> Network Packet -> Deserialization on Client -> Client-Side UI & Prediction Logic

