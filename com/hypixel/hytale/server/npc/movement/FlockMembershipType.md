---
description: Architectural reference for FlockMembershipType
---

# FlockMembershipType

**Package:** com.hypixel.hytale.server.npc.movement
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum FlockMembershipType implements Supplier<String> {
```

## Architecture & Concepts
FlockMembershipType is a foundational enumeration within the server-side NPC artificial intelligence system. It provides a fixed, compile-time-verified set of constants to represent an entity's role and status within a flocking group. This component is critical for behavior trees, target selectors, and movement controllers that need to query and apply logic based on an NPC's relationship to a flock.

By defining a constrained set of states—Leader, Follower, Member, NotMember, and Any—the system avoids the fragility of string-based or integer-based flags. This design choice eliminates an entire class of bugs related to invalid state representation.

The implementation of the Supplier interface, which exposes the get method, standardizes the way human-readable descriptions are retrieved. This is primarily intended for server logs, debugging consoles, and administrative tools, providing clear context without requiring external mapping logic.

## Lifecycle & Ownership
- **Creation:** All instances of this enum (Leader, Follower, etc.) are constructed by the Java Virtual Machine during class loading. They are effectively static singletons managed by the runtime.
- **Scope:** Application-level. Once the FlockMembershipType class is loaded, its constants are available for the entire lifetime of the server process.
- **Destruction:** The enum constants are destroyed only when the server application shuts down and its class loader is garbage collected.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a private final String field for its description, which is set at compile time and cannot be altered. The object's state is fixed for its entire lifecycle.
- **Thread Safety:** **Inherently thread-safe**. As immutable, JVM-managed singletons, these constants can be safely accessed, passed, and compared across any number of threads without locks or other synchronization primitives. This is critical for the multi-threaded server environment where AI ticks may occur concurrently.

## API Surface
The primary API consists of the enum constants themselves. The methods provide supplementary data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | String | O(1) | Returns the human-readable description of the enum constant. |

## Integration Patterns

### Standard Usage
This enum should be used for direct, type-safe comparison to determine NPC behavior within the movement and AI systems. It is commonly retrieved from an NPC's flocking component.

```java
// Example from a hypothetical NPC behavior update method
FlockingComponent flocking = npc.getComponent(FlockingComponent.class);
FlockMembershipType membership = flocking.getMembershipType();

// Use direct object comparison for maximum performance and safety
if (membership == FlockMembershipType.Leader) {
    // Execute flock leadership logic, like setting a new destination
    updateFlockWaypoint();
} else if (membership == FlockMembershipType.Follower) {
    // Execute logic to follow the leader
    followFlockLeader();
}
```

### Anti-Patterns (Do NOT do this)
- **String-Based Comparison:** Never use the output of the get method for logical comparisons. This is slow, error-prone, and defeats the purpose of a type-safe enum.
    ```java
    // ANTI-PATTERN: Brittle and inefficient
    if (membership.get().equals("Is leader of a flock")) {
        // ...
    }
    ```
- **Null Checks:** An entity's flocking status should always resolve to a valid constant, typically NotMember by default. Code should not be written to expect a null FlockMembershipType. A null value indicates a severe bug in the component initialization logic.
- **Instantiation via Reflection:** Attempting to create new instances of this enum using reflection will break the JVM's singleton guarantee and lead to unpredictable behavior.

## Data Pipeline
FlockMembershipType does not process data; it represents a state used in decision-making. Its role is to direct data flow within the AI system.

> **Decision Flow:**
> AI Behavior Tick -> Query Entity's FlockingComponent -> Retrieve **FlockMembershipType** State -> Behavior Tree Node Evaluation -> Select Movement Strategy (e.g., Follow, Lead, Wander) -> Update Entity Velocity and Position

