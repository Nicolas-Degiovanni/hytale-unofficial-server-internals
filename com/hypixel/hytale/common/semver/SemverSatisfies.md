---
description: Architectural reference for SemverSatisfies
---

# SemverSatisfies

**Package:** com.hypixel.hytale.common.semver
**Type:** Contract / Behavioral Interface

## Definition
```java
// Signature
public interface SemverSatisfies {
```

## Architecture & Concepts
The SemverSatisfies interface defines a formal contract for evaluating semantic versions against a specific set of rules. It is a core component of the engine's dependency and compatibility checking system.

Architecturally, this interface embodies the **Strategy Pattern**. It decouples the client code that needs to perform a version check (e.g., a mod loader, a plugin manager) from the concrete logic of the check itself. This allows for a flexible and extensible system where new types of version constraints (e.g., exact matches, caret ranges, greater-than-or-equal-to) can be introduced without modifying the client code.

An object implementing this interface represents a single, specific version requirement. The primary consumer of this interface is any system that must validate a component's version before loading or enabling it.

### Lifecycle & Ownership
As an interface, SemverSatisfies does not have a lifecycle of its own. The lifecycle pertains to the objects that implement it.

- **Creation:** Implementations are typically instantiated by configuration parsers or dependency managers. For example, when parsing a mod's metadata file that specifies `version = "1.10.x"`, a corresponding implementation of SemverSatisfies would be created to encapsulate that rule.
- **Scope:** The lifetime of an implementing object is tied to the component whose version requirement it represents. It generally persists as long as the configuration or metadata is held in memory.
- **Destruction:** Instances are managed by the Java Garbage Collector. No manual cleanup is required, as implementations are expected to be lightweight and hold minimal state.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Implementations are expected to be **effectively immutable**. Their internal state, which defines the version rule, should be established at construction time and not change thereafter.
- **Thread Safety:** All implementations of this interface **must be thread-safe**. The satisfies method should be a pure function with no side effects, allowing it to be called concurrently from multiple threads without locks or synchronization. This is critical for parallelized asset or mod loading systems.

## API Surface
The public contract consists of a single method for evaluating a version.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| satisfies(Semver version) | boolean | O(1) | Evaluates if the provided Semver instance meets the criteria defined by the implementation. Returns true if satisfied, false otherwise. |

## Integration Patterns

### Standard Usage
The primary pattern is to obtain an implementation from a configuration source and use it to validate a target version. Developers should not typically implement this interface directly unless creating a new type of version constraint logic.

```java
// Assume 'dependencyRule' is an object loaded from a manifest
// that implements SemverSatisfies.
SemverSatisfies dependencyRule = modManifest.getEngineVersionRequirement();

// Get the current engine version
Semver currentEngineVersion = Engine.getVersion();

// Check for compatibility
if (!dependencyRule.satisfies(currentEngineVersion)) {
    throw new IncompatibleEngineVersionException("Mod requires a different engine version.");
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create implementations where the version rule can change after construction. This violates the expected immutable nature of the contract and can lead to non-deterministic behavior in multi-threaded contexts.
- **Null Arguments:** Passing a null Semver object to the satisfies method is considered a contract violation. While some implementations may perform a null check, many will throw a NullPointerException. The caller is responsible for ensuring the argument is non-null.

## Data Pipeline
The SemverSatisfies interface acts as a predicate in a larger data validation pipeline, typically transforming a version object into a boolean decision.

> Flow:
> Configuration Source (e.g., JSON manifest) -> Parser -> **SemverSatisfies (Implementation)** -> Version Check Logic -> Boolean Result -> System Action (e.g., Load Component / Log Warning)

