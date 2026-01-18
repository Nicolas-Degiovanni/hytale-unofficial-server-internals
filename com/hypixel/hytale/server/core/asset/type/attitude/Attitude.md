---
description: Architectural reference for Attitude
---

# Attitude

**Package:** com.hypixel.hytale.server.core.asset.type.attitude
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum Attitude implements Supplier<String> {
```

## Architecture & Concepts
The Attitude enum is a fundamental data type that defines the disposition of one game entity towards another. It serves as a core component of the server-side Artificial Intelligence (AI) and Faction systems. Rather than being a complex service, Attitude provides a constrained, type-safe vocabulary for expressing relationships, which directly drives entity behavior.

This enumeration is the primary mechanism for an AI state machine to determine its next action. For example, an entity's decision to attack, flee, or ignore another entity is a direct result of evaluating the Attitude between them. It acts as a contract between different game logic systems, ensuring that concepts like hostility or friendliness are represented consistently throughout the server codebase.

The inclusion of a static EnumCodec field indicates that this type is designed for persistence and network serialization, allowing an entity's relational state to be saved or transmitted to clients.

## Lifecycle & Ownership
As a Java enum, Attitude has a lifecycle strictly managed by the Java Virtual Machine (JVM), not by any application-level framework or factory.

- **Creation:** All enum constants (IGNORE, HOSTILE, etc.) are instantiated by the JVM when the Attitude class is first loaded. This is a one-time, automatic process that occurs early in the server's startup sequence.
- **Scope:** The enum constants are static singletons with a global, application-wide scope. A single instance of Attitude.HOSTILE exists for the entire lifetime of the server process.
- **Destruction:** The constants are reclaimed by the garbage collector only when the server application shuts down and its class loader is unloaded. There is no manual destruction or cleanup process.

## Internal State & Concurrency
- **State:** The Attitude enum is deeply immutable. Each constant holds a final `description` field that is set at compile time and cannot be changed. The static `CODEC` and `VALUES` fields are also final, providing a stable, read-only contract.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutable nature and the JVM's guarantees for enum instantiation, all constants and static fields can be safely accessed from any thread without synchronization. It is a foundational data type designed for high-concurrency environments like game servers.

## API Surface
The public contract is minimal, focusing on data access and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | String | O(1) | Returns the human-readable description for the attitude. |
| CODEC | EnumCodec | O(1) | A static codec for serializing and deserializing the enum, likely by name. |
| VALUES | Attitude[] | O(1) | A static, cached array of all possible Attitude constants. |

## Integration Patterns

### Standard Usage
Attitude is intended to be used for direct comparison in game logic to make behavioral decisions. It is retrieved from faction or AI systems, not instantiated directly.

```java
// How a developer should normally use this
Attitude currentAttitude = FactionSystem.getAttitude(entityA, entityB);

if (currentAttitude == Attitude.HOSTILE) {
    entityA.getAIController().setTarget(entityB);
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal-Based Logic:** Do not use the `ordinal()` method for persistence or network serialization. The static `CODEC` field should be used instead. Relying on the declaration order is extremely brittle and will break saved data or network compatibility if constants are reordered.
- **Reflection:** Do not attempt to create new instances of this enum via reflection. This violates the singleton guarantee of enums and can lead to unpredictable and unstable server behavior.
- **Null Checks:** Logic should be written to handle a valid Attitude state. A system returning a null Attitude represents a failure in the upstream logic (e.g., an unconfigured faction relationship) and should be treated as an error.

## Data Pipeline
Attitude is not a processing stage in a pipeline; it is the *data* that flows through decision-making pipelines within the AI and Faction systems.

> Flow:
> Faction System Evaluation -> **Attitude** (Result) -> AI Behavior Tree -> Entity Action (e.g., Attack, Flee)

