---
description: Architectural reference for BuilderDescriptor
---

# BuilderDescriptor

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Data Model / Descriptor

## Definition
```java
// Signature
public class BuilderDescriptor {
```

## Architecture & Concepts

The BuilderDescriptor is a foundational metadata container that defines the schema for a configurable game asset, specifically an NPC. It does not represent an actual NPC instance, but rather the blueprint from which instances can be constructed and validated. This class is a cornerstone of the server's data-driven design, allowing new NPC types to be defined in external configuration files without requiring changes to the core engine code.

In the broader architecture, BuilderDescriptor acts as a contract. A central registry, likely the NpcAssetRegistry, consumes these descriptors during server initialization. Later, when game logic requests the creation of an NPC, the registry uses the corresponding BuilderDescriptor to:
1.  Understand the required attributes and their types.
2.  Provide metadata to authoring tools or user interfaces.
3.  Execute a series of validation rules to ensure the configured NPC is in a legal state.
4.  Run provider evaluators to dynamically resolve or compute asset properties.

This decouples the static definition of an asset type from the dynamic logic of its instantiation, promoting modularity and content extensibility.

### Lifecycle & Ownership
-   **Creation:** Instances are created by a high-level asset loading system during server bootstrap or when a content pack is loaded. The loader parses a configuration file (e.g., HOCON, JSON) and populates a new BuilderDescriptor instance with the defined schema.
-   **Scope:** A BuilderDescriptor is a long-lived object. Once created and registered, it persists for the entire server session. It is considered part of the server's static, immutable knowledge base.
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down or if its associated content pack is dynamically unloaded.

## Internal State & Concurrency
-   **State:** The object's state is a hybrid of immutable and mutable. Core properties like *name* and *category* are final and set at construction. However, the internal lists of attributes, validators, and evaluators are mutable and are populated via the public *add* methods.

    **WARNING:** After the initial loading and population phase, a BuilderDescriptor instance must be treated as **effectively immutable**. Any modification after it has been registered with a system service will lead to undefined behavior and severe data consistency issues.

-   **Thread Safety:** This class is **not thread-safe**. The internal collections, such as ObjectArrayList, are not synchronized. All population methods (addAttribute, addValidator) must be called from the same thread that constructs the object, typically the main server thread during the asset loading phase. Concurrent access during this population phase will corrupt the descriptor's state. Read-only access from multiple threads after population is safe.

## API Surface

The public API is designed for the construction and population of the descriptor. Getters for retrieving the configured data are assumed to exist but are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addAttribute(attribute) | BuilderAttributeDescriptor | O(1) amortized | Appends a new attribute schema to the descriptor. |
| addValidator(validator) | void | O(1) amortized | Appends a validation rule to be executed against asset instances. |
| addProviderEvaluator(evaluator) | void | O(1) amortized | Appends a dynamic property evaluator to the descriptor. |

## Integration Patterns

### Standard Usage

A BuilderDescriptor should be instantiated and fully configured within a dedicated asset loading or registration module. The typical pattern involves creating the object and then immediately populating its collections before passing it to a central registry.

```java
// Executed within an AssetLoader during server initialization
BuilderDescriptor descriptor = new BuilderDescriptor(
    "goblin_shaman",
    "goblin_tribe",
    "A magic-wielding goblin.",
    "...",
    tags,
    BuilderDescriptorState.STABLE
);

// Populate attributes from config
descriptor.addAttribute("health", "integer", ...);
descriptor.addAttribute("staff_type", "item_ref", ...);

// Add validation logic
descriptor.addValidator(new HealthValidator(10, 100));

// Register the completed descriptor
npcAssetRegistry.register(descriptor);
```

### Anti-Patterns (Do NOT do this)
-   **Modification After Registration:** Never retain a reference to a BuilderDescriptor and modify it after it has been passed to a registry. This violates the "effectively immutable" contract and can cause inconsistent NPC generation across the server.
-   **Concurrent Population:** Do not populate a descriptor from multiple threads. The internal collections are not synchronized and will become corrupted.
-   **Partial Initialization:** Do not register a partially configured descriptor. All attributes, validators, and evaluators must be added during the initial loading phase.

## Data Pipeline

The BuilderDescriptor is a key component in the asset definition pipeline, not a runtime data processing pipeline. It translates static configuration into an in-memory object model that the engine can use.

> Flow:
> Asset Configuration File (e.g., npc/goblin.hocon) -> HOCON Parser -> AssetLoader -> **BuilderDescriptor** -> NpcAssetRegistry

