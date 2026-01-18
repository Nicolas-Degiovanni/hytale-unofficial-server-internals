---
description: Architectural reference for BlockEventType
---

# BlockEventType

**Package:** com.hypixel.hytale.server.npc.blackboard.view.event.block
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum BlockEventType implements Supplier<String> {
```

## Architecture & Concepts
BlockEventType defines a closed, compile-time-safe set of identifiers for events related to world blocks. Its primary architectural function is to eliminate "magic strings" and primitive constants when classifying block-related occurrences, thereby increasing system robustness and readability.

This enumeration is a foundational component of the server-side NPC AI system, specifically within the *Blackboard* subsystem. The Blackboard acts as an NPC's sensory memory, and it uses these event types as keys to trigger reactive behaviors. For example, an NPC might register a listener for the DESTRUCTION event to know when a player has mined a nearby wall, potentially altering its pathfinding or alert state.

By implementing the Supplier interface, BlockEventType provides a standardized contract for retrieving a human-readable description. This is leveraged by debugging tools, server logs, and potentially in-game development consoles to provide clear, descriptive information about AI triggers.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically and exclusively by the Java Virtual Machine (JVM) during class loading. This process is guaranteed to occur only once per application lifecycle.
- **Scope:** All defined constants (DAMAGE, DESTRUCTION, INTERACTION) are static, final, and persist for the entire lifetime of the server application. They are effectively global, immutable singletons.
- **Destruction:** Instances are reclaimed by the JVM during final application shutdown when the corresponding classloader is garbage collected. Manual memory management is neither possible nor required.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a private, final String field for its description, which is set at creation time and can never be modified. The public VALUES array is also static and final, providing a stable, unchanging collection for iteration.
- **Thread Safety:** **Inherently thread-safe**. As immutable singletons managed by the JVM, BlockEventType constants can be safely accessed, passed, and compared across any number of threads without requiring locks or any other synchronization primitives.

## API Surface
The primary API consists of the constants themselves, which are used for type identification and comparison.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DAMAGE | BlockEventType | O(1) | Singleton constant representing a block taking damage but not being destroyed. |
| DESTRUCTION | BlockEventType | O(1) | Singleton constant representing the complete destruction and removal of a block. |
| INTERACTION | BlockEventType | O(1) | Singleton constant representing a non-destructive use action on a block, such as flipping a lever. |
| get() | String | O(1) | Returns the human-readable description of the event type. Fulfills the Supplier contract. |
| VALUES | BlockEventType[] | O(1) | A cached, static array of all enum constants. Use for efficient iteration. |

## Integration Patterns

### Standard Usage
BlockEventType is intended to be used for event dispatching, filtering, and in control flow statements like switch blocks. The most common pattern involves an NPC registering a behavior with its Blackboard that triggers on a specific event type.

```java
// Standard pattern for registering an AI behavior to a block event
Blackboard blackboard = npc.getBlackboard();

// The NPC will now execute this logic whenever a nearby block is destroyed
blackboard.on(BlockEventType.DESTRUCTION, (event) -> {
    npc.getBehaviorController().investigate(event.getSourcePosition());
});
```

### Anti-Patterns (Do NOT do this)
- **String Comparison:** Never compare an event type to a raw string. This defeats the entire purpose of a type-safe enum and is highly error-prone.
    - **BAD:** `if (eventType.get().equals("On block destruction")) { ... }`
    - **GOOD:** `if (eventType == BlockEventType.DESTRUCTION) { ... }`
- **Extensibility via Inheritance:** Enums in Java are final and cannot be extended. This is by design to guarantee a closed, known set of constants. If new event types are required, the enum source code must be modified directly.
- **Reflection Abuse:** Do not use reflection to access private fields or attempt to create new instances of the enum. This violates JVM guarantees and will lead to unpredictable and unstable system behavior.

## Data Pipeline
BlockEventType does not process data itself; it acts as a static data *classifier*. It is attached to event objects to give them semantic meaning as they flow through the game's event bus and AI systems.

> Flow:
> Low-Level Engine Action (e.g., Player swings pickaxe) -> World Block System (calculates damage) -> Event Factory (creates BlockEvent object) -> **BlockEventType.DAMAGE** (assigned to event) -> Event Bus -> NPC Blackboard (subsystem receives event) -> AI Behavior (trigger)

