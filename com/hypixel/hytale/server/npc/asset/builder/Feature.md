---
description: Architectural reference for Feature
---

# Feature

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Utility

## Definition
```java
// Signature
public enum Feature implements Supplier<String> {
```

## Architecture & Concepts
The Feature enum is a foundational component of the server-side NPC asset building system. It provides a type-safe, compile-time-verified set of constants representing all possible target types for an NPC's attention or action. This includes other entities, specific world positions, or abstract concepts like a navigation path.

Architecturally, this enum serves as a schema for defining NPC behavior. By using an enum instead of raw strings or integers, the system prevents a large class of configuration errors and simplifies the logic within the AI behavior tree.

The static **EnumSet** fields, such as AnyPosition and LiveEntity, are a critical design choice. They represent pre-computed, highly optimized collections for common category checks. Higher-level systems, such as target acquisition or threat assessment, can use these sets to perform efficient filtering without iterating through multiple individual checks. For example, determining if a target is a living creature is a simple `LiveEntity.contains(feature)` check, which is significantly more performant and readable than a series of OR conditions.

## Lifecycle & Ownership
- **Creation:** All enum instances (Player, NPC, etc.) are constructed automatically by the Java Virtual Machine during class loading when the Feature enum is first referenced. This process is managed entirely by the JVM.
- **Scope:** As static final instances, all Feature constants are application-scoped. They exist for the entire lifetime of the server process once loaded.
- **Destruction:** The instances are garbage collected only when the JVM shuts down. There is no manual memory management or destruction required.

## Internal State & Concurrency
- **State:** The Feature enum is **immutable**. Each constant has a final `description` field initialized at creation. The static EnumSet fields are initialized once during class loading and are not intended to be modified at runtime.

- **Thread Safety:** This class is **fully thread-safe**. The JVM guarantees that enum constants are created and initialized in a thread-safe manner. All fields are final, and reading them from multiple threads concurrently is inherently safe.

    **WARNING:** The static EnumSet fields (AnyPosition, AnyEntity, LiveEntity) are not defensively copied and are technically mutable. Modifying these sets at runtime via reflection or other means is a severe anti-pattern that would lead to unpredictable behavior across the entire NPC AI system. They must be treated as read-only collections.

## API Surface
The primary API consists of the enum constants themselves and the static sets for categorization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Player, NPC, Drop, Position, Path | Feature | O(1) | Represents a specific, constant target type. |
| AnyPosition | EnumSet | O(1) | A static set for checking if a Feature represents any form of positional target. |
| AnyEntity | EnumSet | O(1) | A static set for checking if a Feature represents any entity-based target. |
| LiveEntity | EnumSet | O(1) | A static set for checking if a Feature represents a living entity (Player or NPC). |
| get() | String | O(1) | Returns the human-readable description of the feature. Fulfills the Supplier contract. |

## Integration Patterns

### Standard Usage
Feature is used to configure NPC behaviors, typically when loading assets from disk or defining behavior tree nodes in code. The static sets are used for efficient type checking.

```java
// Example: Configuring a behavior to target players
BehaviorConfig config = new BehaviorConfig();
config.setTargetType(Feature.Player);

// Example: Checking if a feature is a valid entity target
Feature currentTarget = getTargetFromAI();
if (Feature.AnyEntity.contains(currentTarget)) {
    // Logic for handling entity targets...
}
```

### Anti-Patterns (Do NOT do this)
- **String Comparison:** Never rely on the string description for logic. Using `feature.get().equals("player target")` defeats the purpose of type safety and is brittle. Always compare the enum constants directly: `feature == Feature.Player`.
- **Modifying Static Sets:** Do not attempt to add or remove elements from the static EnumSet fields. These are considered globally constant and modifying them will break AI logic assumptions system-wide.
- **Extensibility via Inheritance:** Enums are final and cannot be extended. To add new target types, the enum itself must be modified, ensuring all dependent systems are aware of the new constant.

## Data Pipeline
The Feature enum acts as a definitional constant within the NPC data pipeline. It does not process data itself but rather provides the schema and categorization for data that flows through the AI system.

> Flow:
> NPC Behavior Asset (JSON/YAML) -> Asset Deserializer -> **Feature** (as a field on a configuration object) -> Behavior Tree Builder -> AI Runtime (for target validation)

