---
description: Architectural reference for SubTypeTypeAdapterFactory
---

# SubTypeTypeAdapterFactory

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class SubTypeTypeAdapterFactory implements TypeAdapterFactory {
```

## Architecture & Concepts
The SubTypeTypeAdapterFactory is a specialized component within the Gson serialization framework. Its primary architectural role is to enable polymorphic serialization for a designated object hierarchy. When serializing an object graph that contains a base class or interface with multiple concrete implementations, this factory injects a type discriminator field into the JSON output. This allows consumers of the JSON to identify which specific subclass the object represents.

This factory is a configuration component, designed to be registered with a GsonBuilder during application bootstrap. It operates by intercepting requests to create a TypeAdapter for its configured base class. Upon interception, it generates a custom, write-only TypeAdapter that performs two key functions:

1.  It adds a specified property (e.g., "type") to the JSON output, with a value corresponding to the registered name of the concrete subclass.
2.  It merges the fields of the original object into the top level of the new JSON object.

**WARNING:** This implementation is strictly for serialization (writing Java objects to JSON). The generated TypeAdapter explicitly throws an UnsupportedOperationException for any read (deserialization) attempts. It is a one-way data flow component.

### Lifecycle & Ownership
-   **Creation:** Instantiated via the static factory method SubTypeTypeAdapterFactory.of. This is followed by a series of chained calls to registerSubType to configure the type mappings. This entire process typically occurs once during the construction of a central Gson instance.
-   **Scope:** The factory's lifecycle is bound to the Gson instance it is registered with. Once the Gson object is built, the factory is held internally by Gson and is used on-demand to create TypeAdapters. It persists for the lifetime of the parent Gson object.
-   **Destruction:** The factory is eligible for garbage collection when the Gson instance that holds a reference to it is dereferenced and collected. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains an internal map, classToName, which stores the mappings from concrete Java classes to their string identifiers. This state is built up during the configuration phase via the registerSubType method. After being passed to a GsonBuilder, this state should be considered immutable.

-   **Thread Safety:** The class is **not thread-safe** during its configuration phase. Calls to registerSubType are not synchronized and should only be performed from a single thread before the factory is used to build a Gson instance. Once the Gson object is created, the factory's internal state is only read, and the create method is safe for concurrent use by multiple threads, as is standard for Gson internals.

    **WARNING:** Modifying the factory's state by calling registerSubType after it has been used to build a Gson instance will lead to undefined behavior and potential race conditions.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(Class, String) | SubTypeTypeAdapterFactory | O(1) | Static factory method. The sole entry point for creating a new factory instance. |
| registerSubType(Class, String) | SubTypeTypeAdapterFactory | O(1) | Registers a concrete subclass and its unique string identifier. Throws IllegalArgumentException if the class or name is a duplicate. |
| create(Gson, TypeToken) | TypeAdapter | O(N) | Core Gson integration point. Called internally by Gson to create the custom TypeAdapter. N is the number of registered subtypes. |

## Integration Patterns

### Standard Usage
The factory must be configured and registered with a GsonBuilder to become active. This is typically done in a centralized location where application-wide services are configured.

```java
// Example: Base class 'Validator' with two implementations.
// 1. Create and configure the factory
SubTypeTypeAdapterFactory validatorFactory = SubTypeTypeAdapterFactory.of(Validator.class, "type")
    .registerSubType(StringLengthValidator.class, "stringLength")
    .registerSubType(NumberRangeValidator.class, "numberRange");

// 2. Register the factory with a GsonBuilder
Gson gson = new GsonBuilder()
    .registerTypeAdapterFactory(validatorFactory)
    .create();

// 3. Use the resulting Gson instance to serialize objects
String jsonOutput = gson.toJson(new StringLengthValidator(10, 20));
// jsonOutput will contain: { "type": "stringLength", "min": 10, "max": 20 }
```

### Anti-Patterns (Do NOT do this)
-   **Deserialization Attempts:** Do not use a Gson instance configured with this factory to deserialize JSON. It will fail with a RuntimeException. This factory is for writing only.
-   **Post-Build Configuration:** Do not call registerSubType on the factory after it has been passed to a GsonBuilder and the Gson object has been created. The behavior is not guaranteed.
-   **Missing Registrations:** Attempting to serialize a subclass of the base type that was not registered via registerSubType will result in an IllegalArgumentException at runtime.

## Data Pipeline
This component acts as a transformation step within the Gson serialization pipeline. It intercepts a specific object type and enriches its JSON representation.

> Flow:
> Java Object (e.g., StringLengthValidator) -> `Gson.toJson()` -> **SubTypeTypeAdapterFactory** -> Custom TypeAdapter -> Enriched JsonObject (with "type" field) -> Final JSON String

