---
description: Architectural reference for FeedbackConsumer
---

# FeedbackConsumer

**Package:** com.hypixel.hytale.server.core.prefab.selection.standard
**Type:** Behavioral Contract / Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface FeedbackConsumer {
   FeedbackConsumer DEFAULT = (key, total, count, target, componentAccessor) -> {};

   void accept(@Nonnull String var1, int var2, int var3, @Nonnull CommandSender var4, @Nonnull ComponentAccessor<EntityStore> var5);
}
```

## Architecture & Concepts
FeedbackConsumer is a functional interface that defines a behavioral contract for handling the result of a server-side operation, typically one involving entity selection or modification. It acts as a callback or a "sink" for status information that needs to be relayed to the originator of an action, represented by a CommandSender.

Architecturally, this interface decouples high-level systems (like command processors or prefab applicators) from the specific implementation of user feedback. Instead of a system being hardcoded to send a chat message, it accepts a FeedbackConsumer. This allows the feedback mechanism to be defined at the call site, enabling different responses for the same core logic. For example, a command executed by a player might send a chat message, while the same logic triggered by an automated system might log to the console or do nothing.

This design embodies the **Strategy Pattern**, where the algorithm for providing feedback is encapsulated in an object and can be swapped out as needed. The static DEFAULT instance provides a null object pattern, offering a safe, no-op implementation to prevent null pointer exceptions when feedback is not required.

### Lifecycle & Ownership
-   **Creation:** Implementations are almost always created as transient lambda expressions or method references at the point where an operation requiring feedback is invoked. The static DEFAULT instance is created once at class-loading time and is globally available.
-   **Scope:** The lifetime of a lambda-based FeedbackConsumer is typically ephemeral, scoped to the duration of a single method call. It is passed as an argument, used once by the receiving system, and then becomes eligible for garbage collection.
-   **Destruction:** Instances are managed by the Java Garbage Collector. As they are usually stateless and short-lived, their memory footprint is negligible.

## Internal State & Concurrency
-   **State:** The FeedbackConsumer contract is inherently stateless. Implementations should not hold or rely on mutable instance data. While a lambda can close over variables from its enclosing scope, this should be done with caution and is considered an implementation detail, not a feature of the interface.
-   **Thread Safety:** The interface itself provides no thread-safety guarantees. Implementations are expected to be executed on the primary server thread responsible for game logic and command processing. **WARNING:** Performing blocking or long-running operations within a FeedbackConsumer implementation is a critical anti-pattern, as it will stall the calling thread and can severely impact server performance.

## API Surface
The public contract consists of a single abstract method and a static default instance.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(key, total, count, target, accessor) | void | Implementation-dependent | The primary callback method. Invoked by a system to report the outcome of an operation. The parameters provide context: a message key, the total entities processed, the count of entities affected, the target to notify, and an accessor to the entity data store. |
| DEFAULT | FeedbackConsumer | O(1) | A shared, static, no-op implementation. It is the preferred value for operations where no feedback is necessary, preventing the need for null checks. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to provide a lambda expression to a system that performs an operation and accepts a FeedbackConsumer. This allows for concise, inline definition of the feedback behavior.

```java
// A hypothetical system that finds and modifies entities,
// then uses the provided consumer to report the result.
EntityOperationService.applyToSelection(selection, operation, (key, total, count, target, accessor) -> {
    // This logic is executed as a callback after the operation completes.
    target.sendMessage(
        String.format("Successfully affected %d of %d entities for operation '%s'.", count, total, key)
    );
});
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Avoid creating implementations that rely on mutable instance fields. A FeedbackConsumer may be unexpectedly reused, leading to unpredictable behavior.

    ```java
    // DO NOT DO THIS
    class BadConsumer implements FeedbackConsumer {
        private int callCount = 0; // Mutable state
        public void accept(...) {
            this.callCount++; // Unsafe if reused
            target.sendMessage("Called " + this.callCount + " times.");
        }
    }
    ```
-   **Blocking Operations:** Never perform blocking I/O, network requests, or heavy computations within the accept method. This will block the server's main thread.

    ```java
    // DO NOT DO THIS
    EntityOperationService.applyToSelection(selection, operation, (key, total, count, target, accessor) -> {
        // This blocks the entire server thread until the file is written!
        Files.writeString(Paths.get("op_log.txt"), "Log entry...");
    });
    ```

## Data Pipeline
FeedbackConsumer is not a component *within* a data pipeline; rather, it is a terminal endpoint for a logical flow, typically originating from a user command.

> Flow:
> Command Input -> Command Execution System -> Entity Selection & Modification -> **FeedbackConsumer** -> Network Packet (to Client) or Console Log

