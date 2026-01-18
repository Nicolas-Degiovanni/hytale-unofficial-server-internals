---
description: Architectural reference for ProviderEvaluator
---

# ProviderEvaluator

**Package:** com.hypixel.hytale.server.npc.asset.builder.providerevaluators
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface ProviderEvaluator {
   void resolveReferences(BuilderManager var1);
}
```

## Architecture & Concepts
The ProviderEvaluator interface defines a critical contract within the server-side NPC asset construction pipeline. It represents a single, specialized step in a multi-stage process responsible for resolving dependencies and references within raw asset definitions.

This interface is a key component of a **Strategy Pattern**. The core asset building system, orchestrated by the BuilderManager, is decoupled from the specific logic required to evaluate different types of data providers. Each implementation of ProviderEvaluator encapsulates the algorithm for resolving a particular kind of reference, such as a texture path, a model link, or a behavior script.

By delegating reference resolution to concrete ProviderEvaluator implementations, the system remains extensible. New types of asset providers can be introduced without modifying the core BuilderManager, simply by creating a new class that implements this interface.

## Lifecycle & Ownership
As an interface, ProviderEvaluator itself has no lifecycle. The lifecycle described here pertains to its concrete implementations.

- **Creation:** Implementations are instantiated and registered with the BuilderManager during the server's asset system initialization. This is typically managed by a factory or a dependency injection framework responsible for configuring the asset pipeline.
- **Scope:** The lifetime of a ProviderEvaluator instance is tied directly to the BuilderManager that owns it. It persists as long as the asset building system is active.
- **Destruction:** Instances are eligible for garbage collection when the parent BuilderManager is destroyed, typically during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** This interface is inherently stateless. However, concrete implementations may be stateful, potentially caching resolved references or other intermediate data to optimize the build process.
- **Thread Safety:** The contract does not enforce thread safety. Implementations must be assumed to be **not thread-safe** unless explicitly documented otherwise. The BuilderManager is responsible for ensuring that calls to resolveReferences are properly synchronized if the asset build process is parallelized. Accessing a shared, mutable ProviderEvaluator from multiple threads without external locking will lead to race conditions and corrupted asset data.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resolveReferences(BuilderManager) | void | Varies | Resolves internal dependencies for a specific asset provider. This method is expected to mutate the state of the asset being built via the provided BuilderManager. |

## Integration Patterns

### Standard Usage
Implementations of this interface are not intended to be invoked directly by application code. They are part of the internal machinery of the asset builder. The BuilderManager discovers and invokes the appropriate evaluator during its processing cycle.

A developer would typically implement this interface to support a new custom asset type.

```java
// Example of a concrete implementation for a custom provider
public class CustomTextureProviderEvaluator implements ProviderEvaluator {
    @Override
    public void resolveReferences(BuilderManager manager) {
        // Logic to find and resolve texture paths
        // using services available on the manager.
        RawAsset asset = manager.getCurrentAsset();
        String textureRef = asset.getProperty("textureRef");
        ResolvedTexture resolved = manager.getTextureRegistry().find(textureRef);
        manager.setResolvedProperty("texture", resolved);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call the resolveReferences method directly. The BuilderManager orchestrates the build process and provides the necessary context and state. Calling it manually will bypass critical validation and state management steps, leading to unpredictable behavior.
- **Stateful Implementations:** Avoid creating implementations that retain mutable state across multiple, independent build cycles. An evaluator should be idempotent and rely only on the context provided by the BuilderManager for each call.

## Data Pipeline
This component acts as a transformation stage within the broader NPC asset compilation pipeline. It takes an unresolved asset reference and transforms it into a resolved, engine-ready object link.

> Flow:
> Raw Asset Definition (HOCON) -> Parser -> Unresolved Asset Object -> **ProviderEvaluator** -> Resolved Asset Object -> BuilderManager State

