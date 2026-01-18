---
description: Architectural reference for StringParameterProvider
---

# StringParameterProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo.parameterproviders
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface StringParameterProvider extends ParameterProvider {
```

## Architecture & Concepts
The StringParameterProvider interface defines a standardized contract for components that supply a dynamic string value to an NPC's sensory or behavior system. It is a core component of the Strategy design pattern, allowing the game's AI logic to be decoupled from the specific sources of its input data.

This abstraction enables NPC behaviors to be configured with various data providers without altering the core decision-making code. For instance, one NPC might use an implementation that provides the name of its current target, while another might use an implementation that provides the name of a nearby point of interest. The consuming system, such as a behavior tree node or a sensor, interacts only with the StringParameterProvider contract, remaining agnostic to the underlying data source.

This interface extends the base ParameterProvider, indicating its role within a broader family of data provider contracts.

## Lifecycle & Ownership
As an interface, StringParameterProvider itself has no lifecycle. The lifecycle and ownership semantics apply to its concrete implementations.

- **Creation:** Implementations are typically instantiated and configured during the definition of an NPC's AI profile or behavior tree. They are often created by a factory or configuration loader that parses the NPC's definition files.
- **Scope:** The lifetime of a provider instance is tied to the NPC behavior it is configured for. It generally persists as long as the NPC's AI controller is active or until the behavior is reconfigured.
- **Destruction:** The provider object is eligible for garbage collection when the owning AI component or behavior configuration is discarded.

## Internal State & Concurrency
The interface itself is stateless. However, implementations may be stateful.

- **State:** Implementations can range from stateless (e.g., returning a constant string) to highly stateful (e.g., caching a value retrieved from the game world). It is the responsibility of the implementation to manage its state correctly.
- **Thread Safety:** **Warning:** Implementations of this interface are not guaranteed to be thread-safe. The `getStringParameter` method may be invoked from the main server thread or a dedicated AI worker thread. Implementers are responsible for ensuring thread safety if their internal state is mutable and accessed from multiple threads. Avoid performing blocking operations within implementations, as this can severely impact server performance.

## API Surface
The public contract consists of a single method for retrieving the string parameter.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getStringParameter() | @Nullable String | Varies | Retrieves the string value from the underlying source. The complexity is implementation-dependent. |

**Nullability Warning:** The contract explicitly allows for a null return value. Consuming systems **must** perform null checks to prevent NullPointerExceptions.

## Integration Patterns

### Standard Usage
The primary integration pattern involves an AI system, such as a sensor or behavior node, holding a reference to a StringParameterProvider. It invokes `getStringParameter` to fetch data needed for its logic.

```java
// An AI system retrieves its configured provider and uses it.
public class NpcTargetNameSensor {
    private final StringParameterProvider targetNameProvider;

    // Provider is injected via constructor
    public NpcTargetNameSensor(StringParameterProvider provider) {
        this.targetNameProvider = provider;
    }

    public void evaluate() {
        String currentTargetName = targetNameProvider.getStringParameter();
        if (currentTargetName != null) {
            // ... perform logic based on the target's name
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming Non-Null:** Never call `getStringParameter` without handling a potential null return. This is a common source of runtime exceptions in AI logic.
- **Expensive Computations:** Do not implement providers that perform computationally expensive or blocking operations (e.g., complex pathfinding, database queries) directly within `getStringParameter`. This will block the calling AI thread and can cause server-wide performance degradation. Offload heavy work to a separate system and have the provider return a cached result.
- **Ignoring Lifecycle:** Do not design implementations that hold strong references to short-lived game objects without proper cleanup logic. This can lead to memory leaks if the provider outlives the object it references.

## Data Pipeline
This interface acts as a data source, initiating a flow of information into the AI system.

> Flow:
> Game World State (e.g., Entity Properties, Zone Data) -> **Concrete StringParameterProvider Implementation** -> NPC Sensor/Behavior -> AI Decision-Making Engine

