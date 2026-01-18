---
description: Architectural reference for BuilderCombatTargetCollector
---

# BuilderCombatTargetCollector

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents.builders
**Type:** Utility

## Definition
```java
// Signature
public class BuilderCombatTargetCollector extends BuilderBase<ISensorEntityCollector> {
```

## Architecture & Concepts
The BuilderCombatTargetCollector is a factory class that operates within the server-side NPC asset loading and instantiation framework. Its sole responsibility is to construct instances of the CombatTargetCollector, a component used in the NPC sensory system.

This class acts as a bridge between declarative NPC behavior definitions (likely specified in asset files) and the concrete Java objects that execute that behavior. The server's asset system maintains a registry of builders, keyed by the component interface they produce. This builder registers itself as a provider for the ISensorEntityCollector interface, allowing NPC assets to request a "CombatTargetCollector" by name or type, which the framework then resolves to this builder.

It is a critical component for enabling dynamic and data-driven NPC configuration, abstracting the asset definition layer from the underlying Java implementation.

### Lifecycle & Ownership
- **Creation:** Instantiated by the NPC asset framework during server bootstrap. The system scans for classes extending BuilderBase, creates an instance of each, and registers it in a central builder registry.
- **Scope:** The BuilderCombatTargetCollector instance is effectively a singleton, managed by the builder registry. It persists for the entire server session.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection when the builder registry is cleared during server shutdown.

## Internal State & Concurrency
- **State:** This class is stateless and immutable. It contains no instance fields and its methods always return the same or new, unshared objects. Its behavior is determined entirely at compile time.
- **Thread Safety:** The BuilderCombatTargetCollector is inherently thread-safe. Its methods, particularly the `build` method, can be invoked from multiple asset-loading threads concurrently without any risk of race conditions or data corruption. No external locking is required.

## API Surface
The public API is designed for consumption by the NPC asset framework, not for direct use by game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ISensorEntityCollector | O(1) | Factory method. Instantiates and returns a new CombatTargetCollector. This is the primary entry point for the asset system. |
| category() | Class<ISensorEntityCollector> | O(1) | Returns the interface token used to register this builder in the framework's registry. |
| isEnabled(ExecutionContext) | boolean | O(1) | A predicate to determine if this builder should be considered for use. Always returns true, indicating it is universally available. |
| getShortDescription() | String | O(1) | Provides human-readable metadata for tooling and debugging, describing the component it builds. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. It is invoked transparently by the NPC asset loading system. An NPC's behavior asset would declare its need for this component, and the framework handles the lookup and instantiation.

A conceptual view of the framework's interaction:
```java
// PSEUDO-CODE: How the NPC Asset Framework uses this builder

// 1. Framework finds a request for an ISensorEntityCollector in an NPC asset file.
BuilderRegistry registry = server.getNpcBuilderRegistry();
BuilderBase builder = registry.findBuilderFor(ISensorEntityCollector.class, "CombatTargetCollector");

// 2. Framework invokes the build method to get the concrete component instance.
if (builder.isEnabled(executionContext)) {
    ISensorEntityCollector component = builder.build(builderSupport);
    npc.addComponent(component);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance via `new BuilderCombatTargetCollector()`. The server's asset framework is responsible for its lifecycle. Manually creating one circumvents the registration process and serves no purpose.
- **Manual Component Creation:** Do not call the `build` method directly to get a CombatTargetCollector. If an NPC requires this component, it must be defined in the NPC's asset data. Bypassing the asset system breaks the data-driven design principle of the AI engine.

## Data Pipeline
This builder is part of the server's configuration and initialization pipeline, not a real-time data processing pipeline. It translates configuration data into executable code.

> Flow:
> NPC Asset File (JSON/HOCON) -> Server Asset Parser -> Builder Registry Lookup -> **BuilderCombatTargetCollector.build()** -> Instantiated CombatTargetCollector -> Attached to NPC Entity's Component List

