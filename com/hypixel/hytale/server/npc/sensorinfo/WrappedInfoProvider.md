---
description: Architectural reference for WrappedInfoProvider
---

# WrappedInfoProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo
**Type:** Transient

## Definition
```java
// Signature
public class WrappedInfoProvider implements InfoProvider {
```

## Architecture & Concepts
The WrappedInfoProvider is a composite data container that aggregates information from multiple sources within the Non-Player Character (NPC) sensory system. It does not generate information itself; instead, it acts as a proxy and collection for one or more successful `Sensor` detections that occur during a single AI evaluation cycle.

Its primary architectural role is to unify the results of a sensor scan into a single, queryable object that conforms to the `InfoProvider` interface. When an NPC's senses detect multiple entities, points of interest, or environmental states simultaneously, this class is used to hold references to all the `Sensor` instances that triggered.

This design allows higher-level AI components, such as Behaviors or Instructions, to operate on a consistent `InfoProvider` abstraction without needing to know how many underlying sensors were involved. The `getExtraInfo` method exemplifies this by implementing a Chain of Responsibility pattern: it queries each contained `Sensor` sequentially, returning the first valid piece of information it finds. This simplifies the decision-making logic by abstracting away the details of the sensory input.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's core NPC sensory processing system. It is created on-demand when a sensor scan yields one or more positive results that need to be passed to the AI decision-making layer. Developers writing AI scripts or behaviors should never instantiate this class directly.
- **Scope:** Extremely short-lived. A WrappedInfoProvider instance exists only for the duration of a single AI tick or behavior evaluation. It is a transient object used to transfer a snapshot of sensory data from the detection phase to the action phase.
- **Destruction:** The object becomes eligible for garbage collection immediately after the AI instruction that consumes its data has finished executing. No long-term references should ever be held to an instance of this class.

## Internal State & Concurrency
- **State:** Highly mutable. The object is created in an empty or partially-filled state and is populated via methods like `addMatch` and `setPositionMatch`. Its internal lists and references are designed to be modified as the results of a sensor scan are processed.
- **Thread Safety:** **This class is not thread-safe.** It is designed for exclusive use within the single thread that manages a specific NPC's AI update loop. Accessing or modifying an instance from multiple threads will result in race conditions and undefined behavior. Any potential for concurrent access must be managed externally with explicit locking, which would be a significant deviation from its intended use.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getExtraInfo(Class) | E | O(N) | Iterates through all contained sensors, querying each for the requested info type. Returns the first non-null result. |
| passExtraInfo(E) | void | O(1) | Attaches an external `ExtraInfoProvider` to this object. |
| addMatch(Sensor) | void | O(1) | Adds a sensor that has successfully detected a match to the internal collection. |
| clearMatches() | void | O(N) | Removes all sensor matches from the internal collection. |
| setPositionMatch(IPositionProvider) | void | O(1) | Sets a specific position provider, typically representing the primary target or location of interest. |
| getParameterProvider(int) | ParameterProvider | O(1) | **Not Implemented.** Always returns null. This provider does not support indexed parameters. |

## Integration Patterns

### Standard Usage
The WrappedInfoProvider is an internal component of the AI system. A behavior or instruction receives it as a parameter from the engine after a successful sensor evaluation. The logic then queries it to extract context for decision-making.

```java
// Inside an AI Behavior or Instruction
// The 'info' object is provided by the NPC's sensory system.
public void execute(AIContext context, InfoProvider info) {
    // Check if the sensory info contains details about a hostile entity
    HostileInfo hostile = info.getExtraInfo(HostileInfo.class);

    if (hostile != null) {
        // Act on the information about the hostile entity
        Entity target = hostile.getTargetEntity();
        context.getNavigator().setTarget(target);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not cache or store an instance of WrappedInfoProvider in a member variable of a behavior. The data it contains represents a single moment in time and will become stale on subsequent AI ticks.
- **Re-use Without Clearing:** Do not attempt to re-use a WrappedInfoProvider instance across multiple operations without first calling `clearMatches` and `clearPositionMatch`. Failure to do so will cause the AI to make decisions based on outdated sensory data.
- **Manual Instantiation:** Never use `new WrappedInfoProvider()`. The sensory system is responsible for its creation and lifecycle. AI logic should only ever consume it.

## Data Pipeline
The class serves as a temporary data bus, carrying information from the point of detection to the point of action within the server's NPC update loop.

> Flow:
> NPC AI Tick -> Sensor System Scans Environment -> Multiple `Sensor` objects trigger -> **WrappedInfoProvider is created and populated** -> AI `Behavior` receives the provider -> `Behavior` calls `getExtraInfo` to make a decision -> `Behavior` issues a command (e.g., move, attack)

