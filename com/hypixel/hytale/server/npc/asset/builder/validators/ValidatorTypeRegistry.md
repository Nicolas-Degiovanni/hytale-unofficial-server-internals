---
description: Architectural reference for ValidatorTypeRegistry
---

# ValidatorTypeRegistry

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class ValidatorTypeRegistry {
```

## Architecture & Concepts
The ValidatorTypeRegistry is a static utility class that serves as a central configuration point for the server's JSON deserialization engine, specifically for handling NPC asset validation rules. It solves the problem of converting declarative validation rules defined in JSON files into executable Java objects.

This class implements a polymorphic deserialization pattern. NPC assets can define a list of validators, where each validator in the JSON has a **Type** field (e.g., "IntRange", "StringNotEmpty"). The ValidatorTypeRegistry maps these string identifiers to concrete Java classes that implement the Validator interface.

It operates by creating and configuring a custom Gson SubTypeTypeAdapterFactory. This factory is then registered with the server's primary Gson instance during its initialization phase. When the asset loader later parses an NPC definition, the configured Gson instance uses this factory to dynamically instantiate the correct Validator implementation based on the **Type** string found in the JSON. This architecture decouples the asset format from the engine's validation logic, allowing new validation rules to be added by simply registering a new class here, without changing the asset parsing code.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. It is a pure utility with only static members. Its `registerTypes` method is invoked during the server's bootstrap sequence, typically as part of configuring the global Gson service.
- **Scope:** The configuration applied by this class persists for the lifetime of the `Gson` object it helps build. This is almost always the entire server session.
- **Destruction:** The registered `TypeAdapterFactory` is destroyed and garbage collected along with the `Gson` instance when the server shuts down.

## Internal State & Concurrency
- **State:** The ValidatorTypeRegistry is entirely stateless. It contains no fields and its behavior is consistent across all calls. The state it produces is injected into the `GsonBuilder` instance passed to it.
- **Thread Safety:** The `registerTypes` method is re-entrant and thread-safe. However, the `GsonBuilder` object it modifies is not thread-safe. Standard usage involves a single thread configuring the builder during application startup, which avoids any concurrency issues.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerTypes(GsonBuilder) | GsonBuilder | O(1) | Registers all known Validator subtypes with a custom `TypeAdapterFactory`. This enables polymorphic deserialization of Validator objects from JSON. Returns the builder for call chaining. |

## Integration Patterns

### Standard Usage
This class should only be called during the initial setup of a `Gson` instance that will be used for loading game assets.

```java
// Executed once during server startup
GsonBuilder builder = new GsonBuilder();

// Register the polymorphic Validator types
ValidatorTypeRegistry.registerTypes(builder);

// Register other factories or configurations...

// Create the immutable, thread-safe Gson instance
Gson assetDeserializer = builder.create();

// This instance is now capable of deserializing Validator definitions
// and should be stored in a central service registry.
```

### Anti-Patterns (Do NOT do this)
- **Manual Registration:** Do not manually register individual `Validator` subtypes using `gsonBuilder.registerTypeAdapter` elsewhere in the codebase. This class is the single source of truth for validator deserialization. Bypassing it will lead to a fragmented and difficult-to-maintain configuration.
- **Delayed Invocation:** Do not call `registerTypes` on a `GsonBuilder` after the `Gson` instance has already been created. The configuration will have no effect on the existing instance.

## Data Pipeline
The ValidatorTypeRegistry does not process data itself; it configures the system that does. The data pipeline it enables is central to asset validation.

> Flow:
> NPC Asset JSON File -> Asset Loading Service -> Gson Deserializer -> **SubTypeTypeAdapterFactory (Configured by ValidatorTypeRegistry)** -> Instantiated Validator Object Graph

