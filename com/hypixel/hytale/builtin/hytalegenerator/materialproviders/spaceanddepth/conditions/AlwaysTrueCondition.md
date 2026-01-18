---
description: Architectural reference for AlwaysTrueCondition
---

# AlwaysTrueCondition

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.conditions
**Type:** Singleton

## Definition
```java
// Signature
public class AlwaysTrueCondition implements SpaceAndDepthMaterialProvider.Condition {
```

## Architecture & Concepts
The AlwaysTrueCondition is a stateless, singleton implementation of the **Strategy** design pattern, specifically for the world generation subsystem. It conforms to the SpaceAndDepthMaterialProvider.Condition interface, which defines a contract for evaluating whether a material should be placed at a given coordinate based on spatial criteria.

This class represents the simplest possible condition: one that is always met. Its primary role is to serve as a default or fallback rule within a SpaceAndDepthMaterialProvider. By providing a concrete "always true" strategy, it allows the material provider's logic to remain clean, avoiding null checks or special case handling for rules that should apply unconditionally. It effectively acts as a "catch-all" condition when configuring a sequence of material placement rules.

## Lifecycle & Ownership
- **Creation:** The single public static final INSTANCE is instantiated eagerly by the Java Class Loader when the AlwaysTrueCondition class is first referenced. This occurs once during the application's startup phase.
- **Scope:** As a static singleton, its lifetime is tied to the application's Class Loader. It persists for the entire duration of the server or client process.
- **Destruction:** The object is eligible for garbage collection only when the JVM shuts down and its Class Loader is unloaded. For all practical purposes, it is never destroyed during normal operation.

## Internal State & Concurrency
- **State:** This class is **immutable and stateless**. It contains no instance fields and its behavior is constant.
- **Thread Safety:** The class is inherently **thread-safe**. Its stateless nature ensures that concurrent calls from multiple world generation threads will not cause interference or race conditions. No synchronization mechanisms are required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| qualifies(int, int, int, int, int, int, int) | boolean | O(1) | Evaluates the condition for a given block location. Always returns true, ignoring all input parameters. |

## Integration Patterns

### Standard Usage
This component is not retrieved from a service registry. It is used by directly referencing its public static instance when configuring a material provider. It is typically the last condition in a chain of rules to act as a default material.

```java
// Example of configuring a material provider
// (Note: Fictional API for demonstration)
SpaceAndDepthMaterialProvider provider = new SpaceAndDepthMaterialProvider();

// Add a conditional rule first
provider.addRule(Material.WATER, new IsBelowSeaLevelCondition());

// Add a final, unconditional fallback rule
provider.addRule(Material.STONE, AlwaysTrueCondition.INSTANCE);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private to enforce the singleton pattern. Do not attempt to create new instances via reflection, as this violates the design contract and provides no benefit.
- **Redundant Checks:** Do not write code that checks if AlwaysTrueCondition.INSTANCE is null. It is a static final field and is guaranteed to be initialized by the JVM.

## Data Pipeline
AlwaysTrueCondition acts as a terminal decision node within the material selection pipeline of the world generator. It does not process or transform data itself, but rather provides a constant signal to guide the data flow.

> Flow:
> World Generator Voxel Request -> SpaceAndDepthMaterialProvider -> Rule Evaluation -> **AlwaysTrueCondition.qualifies()** -> Returns `true` -> Material Provider Applies Rule -> Pipeline Terminates

