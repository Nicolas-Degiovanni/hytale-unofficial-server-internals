---
description: Architectural reference for BlockStateRegistration
---

# BlockStateRegistration

**Package:** com.hypixel.hytale.server.core.universe.world.meta
**Type:** Transient Data Object

## Definition
```java
// Signature
public class BlockStateRegistration extends Registration {
```

## Architecture & Concepts
The BlockStateRegistration class is a foundational descriptor object within the world metadata system. It does not represent an active, in-world block instance but rather serves as a manifest entry for a specific type of BlockState. Its primary function is to encapsulate the definition of a block type—linking its concrete Java class with lifecycle rules—for submission to a central registry.

This class extends the generic Registration type, inheriting a standardized contract for lifecycle management. This includes an enablement condition (a BooleanSupplier) and an unregistration hook (a Runnable). This design pattern decouples the definition of a block from the system that manages it, allowing for dynamic content loading and unloading, such as mods or feature flags.

In essence, a BlockStateRegistration is a configuration record that tells the server: "Here is a type of block that can exist, here is how to know if it's currently enabled, and here is what to do if it needs to be removed."

### Lifecycle & Ownership
-   **Creation:** Instances are created during the server's content loading phase. This is typically done by systems responsible for defining the game's content, such as the core engine's block loader or a third-party mod initialization service. A new instance is created for each unique BlockState class that needs to be made available to the world engine.
-   **Scope:** The object's lifetime is tied directly to the content it represents. It is held by the central Block Registry for as long as the block type is considered valid. If the underlying content is unloaded (e.g., a mod is disabled), the registration is removed from the registry.
-   **Destruction:** The object becomes eligible for garbage collection once it is removed from the registry and no other systems hold a reference to it. The *unregister* Runnable provided at creation is executed by the registry just before the reference is dropped, allowing for orderly cleanup.

## Internal State & Concurrency
-   **State:** The internal state of a BlockStateRegistration instance is **immutable**. Its core data, the blockStateClass, is a final field set at construction. The parent Registration class manages the lifecycle delegates (isEnabled, unregister), but this object itself does not contain mutable fields. Its perceived state (i.e., whether it is active) is determined by the external BooleanSupplier, not by an internal flag.
-   **Thread Safety:** This class is inherently thread-safe due to its immutability. However, the delegates (BooleanSupplier and Runnable) passed into its constructor are executed by the registry system and are not guaranteed to be called from any specific thread.

    **Warning:** It is the responsibility of the instantiating system to ensure that the provided BooleanSupplier and Runnable are themselves thread-safe. Failure to do so can lead to race conditions during content reloads or concurrent server operations.

## API Surface
The primary interaction with this class is through its constructor and a single public accessor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlockStateClass() | Class<? extends BlockState> | O(1) | Retrieves the concrete BlockState class this registration represents. This is the primary payload of the object. |

## Integration Patterns

### Standard Usage
The intended use is to create an instance and immediately submit it to a registry system. The object acts as a self-contained definition that the registry can then manage.

```java
// Within a content initialization service or mod entry point
BlockRegistry blockRegistry = serverContext.getService(BlockRegistry.class);
FeatureFlagService featureFlags = serverContext.getService(FeatureFlagService.class);

// Create a registration for a new, experimental block type
BlockStateRegistration newBlockReg = new BlockStateRegistration(
    ExperimentalObsidianBlockState.class,
    () -> featureFlags.isEnabled("experimental_blocks"), // Enablement is tied to a feature flag
    () -> log.info("Unregistering ExperimentalObsidianBlockState") // Cleanup logic
);

// Submit the definition to the central registry for management
blockRegistry.register(newBlockReg);
```

### Anti-Patterns (Do NOT do this)
-   **Retaining References:** Do not store a reference to a BlockStateRegistration object after submitting it to a registry. The registry assumes full ownership of the registration's lifecycle. Attempting to interact with the object after submission can interfere with the registry's state management.
-   **Stateful Delegates:** Avoid providing a BooleanSupplier or Runnable that depends on complex, unmanaged, or non-thread-safe external state. These delegates may be invoked at unexpected times or from multiple threads. They should be lightweight and idempotent.

## Data Pipeline
This class is a data container, not a processing node. It serves as an input to the registry, which in turn populates the data used by the world generation and simulation engine.

> Flow:
> Mod/Content Loader -> **new BlockStateRegistration(...)** -> Block Registry -> World Generator -> BlockState Instantiation Logic

