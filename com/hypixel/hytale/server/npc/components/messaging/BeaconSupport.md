---
description: Architectural reference for BeaconSupport
---

# BeaconSupport

**Package:** com.hypixel.hytale.server.npc.components.messaging
**Type:** Component

## Definition
```java
// Signature
public class BeaconSupport extends MessageSupport implements Component<EntityStore> {
```

## Architecture & Concepts
The BeaconSupport component provides a stateful, indexed messaging system for a server-side entity, typically an NPC. It functions as a set of named, activatable flags or signals—referred to as "beacons" or "messages"—that allow an entity to broadcast its internal state to other game systems.

Architecturally, this component serves as a decoupled communication layer. Instead of systems directly querying an NPC's complex internal state (e.g., its current AI goal), they can query the status of a simple, named message like "hasTarget" or "isFleeing". This simplifies inter-system communication and makes NPC behavior more observable.

The core mechanism relies on a mapping between human-readable string identifiers for messages and high-performance integer indices. During initialization, a predefined map of messages is provided, which the component uses to create an array of `NPCMessage` slots. This pre-allocation and integer-based lookup ensures that message posting and polling operations are extremely fast, suitable for use within the main game loop.

This component is managed by the server's core entity-component system and is retrieved via the `NPCPlugin`'s component type registry.

### Lifecycle & Ownership
-   **Creation:** An instance of BeaconSupport is not created directly. It is instantiated and attached to an entity by the server's entity management system when an NPC is spawned. The `clone()` method suggests that new components are created by copying a prototype instance defined in an entity template.
-   **Scope:** The lifecycle of a BeaconSupport instance is strictly tied to the parent entity to which it is attached. It persists as long as the entity exists in the world.
-   **Destruction:** The component is marked for garbage collection when its parent entity is despawned or removed from the `EntityStore`. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** This component is highly mutable. Its primary state is stored in the `messageSlots` array, which holds `NPCMessage` objects. Each `NPCMessage` tracks its activation status, target entity, and age. The `messageIndices` map, used for string-to-integer lookups, is configured once during the `initialise` phase and is treated as immutable thereafter.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be accessed exclusively from the main server thread that ticks the game world and its entities. All methods perform non-atomic reads and writes to shared state (`messageSlots`). Concurrent access from multiple threads will lead to race conditions and corrupt the component's state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| postMessage(message, target, age) | void | O(1) | Activates a message slot by its string name. Throws NullPointerException if called before `initialise`. |
| pollMessage(messageIndex) | Ref<EntityStore> | O(1) | Retrieves the target of an active message and immediately deactivates the slot. Returns null if no target. |
| peekMessage(messageIndex) | Ref<EntityStore> | O(1) | Retrieves the target of an active message without changing its state. Returns null if no target. |
| initialise(messageIndices) | void | O(N) | **CRITICAL:** Initializes the component with a defined set of messages. Must be called once before any other operation. |
| getMessageTextForIndex(index) | String | O(1) | Performs a reverse lookup to get the string name for a given message index. |
| getMessageSlots() | NPCMessage[] | O(1) | Returns a direct reference to the internal message slot array. **Warning:** External modification is not advised. |

## Integration Patterns

### Standard Usage
The component is intended to be retrieved from an entity and used by AI behaviors or other game logic systems to signal state changes. Another system can then query this component to react to those changes.

```java
// In an AI or entity update system
// Assume 'entity' is the NPC we are controlling

BeaconSupport beacon = entity.getComponent(BeaconSupport.getComponentType());

// An event occurs, so we post a message
// The "aggro" message must have been defined during initialization
Ref<EntityStore> playerTarget = findNearbyPlayer();
if (playerTarget != null) {
    // Signal that this NPC is now hostile towards a target
    beacon.postMessage("aggro", playerTarget, 0.0);
}

// Another system (e.g., a quest tracker) might check this state
// Assume 'aggroMessageIndex' was retrieved and cached earlier
Ref<EntityStore> currentTarget = beacon.peekMessage(aggroMessageIndex);
if (currentTarget != null) {
    // Logic to handle the NPC being in an "aggro" state
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BeaconSupport()`. The component is managed by the entity system. A manually created instance will be uninitialized and non-functional.
-   **Access Before Initialization:** Calling `postMessage` or other methods before the `initialise` method has been invoked by the entity factory will result in a `NullPointerException`.
-   **External State Modification:** The `getMessageSlots()` method returns a direct reference to the internal array. Do not modify this array from outside the class, as it can break the component's state guarantees.
-   **Multi-threaded Access:** Do not access a BeaconSupport instance from any thread other than the primary server tick thread for that entity. This will cause unpredictable behavior and state corruption.

## Data Pipeline
BeaconSupport does not process a stream of data; rather, it acts as a stateful controller. The flow is one of command and query.

> Flow:
> AI Behavior Tree or Game Event -> `postMessage("someState", target)` -> **BeaconSupport** (Internal `messageSlots` array is updated) -> External System (e.g., Quest Logic, Special Effects Controller) -> `peekMessage(someStateIndex)` -> State is read and used for decision making.

