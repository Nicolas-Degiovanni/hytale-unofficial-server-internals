---
description: Architectural reference for EmptyBuilderModifier
---

# EmptyBuilderModifier

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Singleton

## Definition
```java
// Signature
public class EmptyBuilderModifier extends BuilderModifier {
```

## Architecture & Concepts
The EmptyBuilderModifier is a specific implementation of the **Null Object Pattern**. It provides a non-null, immutable, "no-op" (no operation) instance of its parent class, BuilderModifier.

Its primary role within the server's NPC asset system is to represent the absence of any modifications. By providing a concrete object instead of a null reference, it allows consuming systems to avoid conditional null checks, leading to cleaner and less error-prone code. Systems that process a collection of BuilderModifier objects can safely include this instance without special handling; its methods are designed to either return "empty" values or signal incorrect usage by throwing an exception.

This class acts as a sentinel value, indicating that a particular asset or component configuration path results in no changes to the NPC builder state.

### Lifecycle & Ownership
- **Creation:** A single static instance, named INSTANCE, is instantiated by the JVM during class loading. The private constructor strictly enforces the singleton pattern.
- **Scope:** Application-wide. The INSTANCE object persists for the entire lifetime of the server process.
- **Destruction:** The object is never explicitly destroyed. It is eligible for garbage collection only when the server application shuts down and its ClassLoader is unloaded.

## Internal State & Concurrency
- **State:** Deeply **immutable**. The object is initialized once with empty and null values passed to its superclass constructor. Its internal state can never be changed after creation.
- **Thread Safety:** Inherently **thread-safe**. As an immutable singleton, the INSTANCE can be safely shared and accessed by any number of threads without requiring locks or other synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isEmpty() | boolean | O(1) | Always returns true, identifying this object as a no-op modifier. |
| exportedStateCount() | int | O(1) | Always returns 0, indicating no component states are managed. |
| applyComponentStateMap(support) | void | O(1) | **Unsupported.** Always throws UnsupportedOperationException. |
| popComponentStateMap(support) | void | O(1) | **Unsupported.** Always throws UnsupportedOperationException. |

## Integration Patterns

### Standard Usage
The EmptyBuilderModifier should be used as a default value or a return value to signify that no modifications are available. Client code should primarily use the isEmpty method or direct instance comparison to gate logic.

```java
// A factory method might return the empty instance as a default
BuilderModifier getModifierFor(NpcConfiguration config) {
    if (config.hasNoOverrides()) {
        return EmptyBuilderModifier.INSTANCE;
    }
    // ... otherwise, build and return a real modifier
}

// Consuming code checks the modifier before attempting to apply it
BuilderModifier modifier = getModifierFor(someConfig);
if (!modifier.isEmpty()) {
    // This logic is safely skipped for the EmptyBuilderModifier
    modifier.applyComponentStateMap(builderSupport);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create a new instance of EmptyBuilderModifier. The constructor is private for this reason. Always use the static EmptyBuilderModifier.INSTANCE field.
- **Attempting Application:** Do not call applyComponentStateMap or popComponentStateMap on this object. These methods will throw an UnsupportedOperationException. Their purpose is to fail fast, indicating a logical error where the calling code did not first check if the modifier was empty.

## Data Pipeline
This class acts as a terminal point or a filter in a data pipeline. When an asset building process encounters this instance, it signifies that the modification pipeline for a given component should halt.

> Flow:
> NPC Asset Request -> Modifier Resolution Logic -> **EmptyBuilderModifier.INSTANCE** -> (Processing for this path terminates)

