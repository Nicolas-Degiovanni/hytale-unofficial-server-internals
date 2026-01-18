---
description: Architectural reference for ProviderEvaluatorTypeRegistry
---

# ProviderEvaluatorTypeRegistry

**Package:** com.hypixel.hytale.server.npc.asset.builder.providerevaluators
**Type:** Utility

## Definition
```java
// Signature
public class ProviderEvaluatorTypeRegistry {
```

## Architecture & Concepts

The ProviderEvaluatorTypeRegistry is a static utility class that serves a single, critical function: configuring the server's JSON serialization system to handle polymorphic types of ProviderEvaluator. It acts as a registration manifest, mapping specific string identifiers found in NPC asset files to concrete Java class implementations.

This class is a key component in the server's data-driven design for Non-Player Characters (NPCs). NPC behaviors and features are defined in external JSON files, not hard-coded in Java. To parse these files, the system uses the Gson library. The ProviderEvaluatorTypeRegistry injects custom deserialization logic into Gson via a SubTypeTypeAdapterFactory. This allows a JSON object with a field like `"Type": "ProvidesFeatureUnconditionally"` to be correctly deserialized into an instance of the UnconditionalFeatureProviderEvaluator class.

Without this registry, the asset loading system would be unable to interpret the varied and extensible provider evaluator definitions, effectively breaking all custom NPC asset loading. It is a foundational piece of the server's asset pipeline.

### Lifecycle & Ownership
-   **Creation:** This class is never instantiated. It is a static utility, and all its members are static.
-   **Scope:** Its methods are available for the entire lifetime of the application's ClassLoader. Its configuration is applied during server initialization and the resulting Gson object persists for the server's session.
-   **Destruction:** Not applicable. The class is unloaded when the JVM shuts down.

## Internal State & Concurrency
-   **State:** The ProviderEvaluatorTypeRegistry is entirely stateless. It contains no fields and does not store or cache any data between calls.
-   **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The `registerTypes` method is a pure function with respect to the class itself.

    **WARNING:** The GsonBuilder object passed as an argument is **not** thread-safe. All configuration of a single GsonBuilder instance, including calls to this class's methods, must be performed by a single thread or be externally synchronized. This is typically handled by performing all Gson configuration during the server's single-threaded bootstrap phase.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerTypes(GsonBuilder) | GsonBuilder | O(1) | Registers all known ProviderEvaluator subtypes with the provided GsonBuilder. This method is idempotent but should only be called once per builder. |

## Integration Patterns

### Standard Usage

This class should be invoked only during the initialization and configuration of the server's primary Gson instance for asset processing. The standard pattern is to chain this call with other type registrations on a single GsonBuilder before it is built.

```java
// Correct usage during server bootstrap or service initialization
GsonBuilder builder = new GsonBuilder();

// Register the provider evaluator types
ProviderEvaluatorTypeRegistry.registerTypes(builder);

// Register other custom types...
// OtherTypeRegistry.registerTypes(builder);

// Finalize the Gson instance
Gson sharedGsonInstance = builder.create();
```

### Anti-Patterns (Do NOT do this)
-   **Late Registration:** Do not call `registerTypes` on a builder after the final Gson object has already been created from it. The registration will have no effect on the existing Gson instance.
-   **Redundant Builders:** Do not create a new GsonBuilder just to call this method. The registration must be applied to the *same* builder instance that will be used to create the globally shared Gson object.

## Data Pipeline

This class does not directly process data. Instead, it configures the system that *will* process the data. It is a critical setup step in the NPC asset loading pipeline.

> Flow:
> NPC Asset JSON on Disk -> I/O Read -> String -> Gson.fromJson() -> **SubTypeTypeAdapterFactory (Configured by ProviderEvaluatorTypeRegistry)** -> Hydrated ProviderEvaluator Java Object

