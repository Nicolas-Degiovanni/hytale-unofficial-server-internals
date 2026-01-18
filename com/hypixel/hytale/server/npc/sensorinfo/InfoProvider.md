---
description: Architectural reference for the InfoProvider interface, a core contract for NPC sensory systems.
---

# InfoProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface InfoProvider {
   @Nullable
   IPositionProvider getPositionProvider();

   @Nullable
   ParameterProvider getParameterProvider(int var1);

   @Nullable
   <E extends ExtraInfoProvider> E getExtraInfo(Class<E> var1);

   <E extends ExtraInfoProvider> void passExtraInfo(E var1);

   @Nullable
   <E extends ExtraInfoProvider> E getPassedExtraInfo(Class<E> var1);

   boolean hasPosition();
}
```

## Architecture & Concepts
The InfoProvider interface defines a fundamental contract within the server-side NPC Artificial Intelligence engine. It serves as a generic, decoupled access point for AI *Sensors* to query contextual information about the game world without being tightly coupled to the specific entity or environment being sensed.

This interface is a key component of a **Dependency Inversion** strategy. Instead of an AI sensor directly accessing a game entity's state (e.g., `npc.getPosition()`), it queries the abstract InfoProvider. This allows the same sensor logic to be reused for different types of entities or even for hypothetical scenarios, as long as a concrete implementation of InfoProvider is supplied.

The methods `passExtraInfo` and `getPassedExtraInfo` suggest a more advanced pattern where information can be passed between different stages of AI processing. One sensor can compute a result, attach it as an ExtraInfoProvider, and a subsequent sensor in the same AI tick can retrieve and use that computed data, preventing redundant calculations.

## Lifecycle & Ownership
As an interface, InfoProvider itself has no lifecycle. The lifecycle pertains to the objects that *implement* this interface.

- **Creation:** Concrete implementations are typically instantiated on-demand by the NPC's AI scheduler or behavior tree just before a sensor evaluation is performed. They are transient, short-lived objects.
- **Scope:** The scope of an InfoProvider implementation is extremely narrow, usually confined to a single AI tick or a single sensor's update cycle. It acts as a temporary container for all the context needed for one specific decision-making process.
- **Destruction:** The object is eligible for garbage collection immediately after the sensor evaluation completes. There is no persistent state held within the provider itself between AI ticks.

**WARNING:** Implementations of this interface should be considered ephemeral. Never cache or hold a reference to an InfoProvider instance across multiple game ticks, as its underlying data may become stale or invalid.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Concrete implementations are stateful by design, but their state is a snapshot of the game world at the moment of their creation. They are effectively immutable for the duration of their short lifecycle.
- **Thread Safety:** Implementations are **not thread-safe** and must not be considered as such. All NPC AI processing is expected to occur on a single, dedicated server thread. Accessing an InfoProvider from an asynchronous task or a different thread will lead to concurrency violations and unpredictable behavior.

## API Surface
The public contract is designed for querying environmental and contextual data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPositionProvider() | IPositionProvider | O(1) | Retrieves a provider for spatial information. Returns null if the context has no position. |
| getParameterProvider(int) | ParameterProvider | O(1) | Retrieves a provider for configured AI parameters, indexed by an integer key. |
| getExtraInfo(Class) | E | O(1) | Retrieves a specialized, type-safe information provider. Used for extending the system with custom data types. |
| passExtraInfo(E) | void | O(1) | Attaches a specialized information provider to this context, allowing data to be passed between sensors. |
| getPassedExtraInfo(Class) | E | O(1) | Retrieves a specialized provider that was previously attached via passExtraInfo. |
| hasPosition() | boolean | O(1) | A fast check to determine if positional data is available, preventing unnecessary calls to getPositionProvider. |

## Integration Patterns

### Standard Usage
The InfoProvider is passed as an argument to an AI sensor's update or evaluation method. The sensor then uses the provider to gather the data it needs to function.

```java
// Inside an AI Sensor's logic
public void evaluate(InfoProvider provider) {
    if (!provider.hasPosition()) {
        // Cannot proceed without a location
        return;
    }

    IPositionProvider posProvider = provider.getPositionProvider();
    Vector3f myPosition = posProvider.getPosition();

    // Use myPosition to perform raycasts, distance checks, etc.
}
```

### Anti-Patterns (Do NOT do this)
- **Casting to Concrete Type:** Never cast an InfoProvider to its concrete implementation. This violates the abstraction and creates a brittle dependency on internal AI systems.
- **Ignoring Nullability:** The `get` methods are annotated with Nullable. Always perform null checks on the returned providers before use. Assuming a provider exists will lead to NullPointerExceptions.
- **Stateful Misuse:** Do not use `passExtraInfo` to persist state across game ticks. The provider is destroyed after the evaluation, and any passed information will be lost.

## Data Pipeline
The InfoProvider acts as a data source within the NPC sensory pipeline. It does not process data itself but rather serves it to the components that do.

> Flow:
> NPC AI Tick -> Behavior Tree Node -> Sensor Evaluation -> **InfoProvider (Implementation Created)** -> Sensor queries InfoProvider for world state -> Sensor produces output -> InfoProvider is discarded.

