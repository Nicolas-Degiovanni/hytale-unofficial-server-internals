---
description: Architectural reference for BuilderDescriptorState
---

# BuilderDescriptorState

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Enumeration

## Definition
```java
// Signature
public enum BuilderDescriptorState {
```

## Architecture & Concepts
The BuilderDescriptorState enum is a metadata component used to classify the development and stability status of an NPC builder configuration. It functions as a type-safe state machine, providing a clear and constrained set of possible statuses for any given NPC asset descriptor.

This enumeration is a critical part of the server's content management and asset pipeline. It allows developers and content creators to version and manage NPC configurations without resorting to fragile "magic strings" or integer flags. The primary architectural role of this enum is to enable conditional logic within the NPC asset loading and instantiation systems, allowing the server to filter out experimental, deprecated, or incomplete NPC definitions from production environments.

## Lifecycle & Ownership
- **Creation:** Instances of this enum (Unknown, WorkInProgress, etc.) are constructed by the Java Virtual Machine (JVM) during class loading. This process is automatic and occurs once.
- **Scope:** As static final constants, these instances exist for the entire lifetime of the server application. They are globally accessible and immutable.
- **Destruction:** The instances are garbage collected only when the class loader itself is unloaded, which typically happens at application shutdown. There is no manual destruction.

## Internal State & Concurrency
- **State:** BuilderDescriptorState is fundamentally immutable. Each enumerated constant represents a fixed, unchangeable value.
- **Thread Safety:** This enum is inherently thread-safe. Its instances can be safely accessed and compared from any thread without synchronization, as their state cannot be modified after creation.

## API Surface
The public contract consists of the defined enumeration constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Unknown | BuilderDescriptorState | O(1) | Default or uninitialized state. Indicates a potential configuration error. |
| WorkInProgress | BuilderDescriptorState | O(1) | The descriptor is actively under development and is likely incomplete or unstable. |
| Experimental | BuilderDescriptorState | O(1) | The descriptor is functionally complete but undergoing testing. Not intended for production use. |
| Stable | BuilderDescriptorState | O(1) | The descriptor is considered complete, tested, and safe for production use. |
| Deprecated | BuilderDescriptorState | O(1) | The descriptor is outdated and scheduled for removal. It should not be used for new development. |

## Integration Patterns

### Standard Usage
This enum is typically used as a field within a larger data structure, such as an NPC descriptor, and evaluated in conditional logic or switch statements to control asset processing.

```java
// How a developer should normally use this
BuilderDescriptor descriptor = assetManager.load("skeletal_warrior.json");

switch (descriptor.getState()) {
    case Stable:
        // Proceed with NPC factory creation
        npcFactory.createFrom(descriptor);
        break;
    case Experimental:
        if (server.isDevelopmentMode()) {
            npcFactory.createFrom(descriptor);
        } else {
            log.warn("Skipping experimental NPC: " + descriptor.getId());
        }
        break;
    case Deprecated:
    case WorkInProgress:
        log.error("Attempted to load an invalid NPC descriptor: " + descriptor.getId());
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Persistence via Ordinal:** Do not serialize or store the integer value from the `ordinal()` method. The numeric order of enum constants is not guaranteed to be stable across code changes. Persisting the string name via the `name()` method is the correct approach.
- **Ignoring New States:** Avoid using a `default` case in switch statements that silently does nothing. This can mask errors when a new state is added to the enum. If a state should not be handled, it is better to explicitly list it or throw an `UnsupportedOperationException` in the default case.

## Data Pipeline
This enum does not process data itself but rather annotates data as it flows through the NPC asset pipeline. It acts as a gatekeeper, influencing whether a descriptor proceeds to the next stage.

> Flow:
> Asset File (JSON) -> Deserializer -> BuilderDescriptor (with **BuilderDescriptorState**) -> Asset Validation Service -> NPC Factory -> Game World

