---
description: Architectural reference for CommandOwner
---

# CommandOwner

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Interface

## Definition
```java
// Signature
public interface CommandOwner {
```

## Architecture & Concepts
The CommandOwner interface represents a fundamental contract within the server's command processing system. It establishes a formal identity for any entity capable of executing a command, such as a Player, the Server Console, or an automated script agent.

This interface serves as a crucial abstraction, decoupling the CommandSystem from the concrete types of command issuers. By interacting with the CommandOwner contract, the system can uniformly handle permissions, logging, and feedback without needing to know the specific details of the entity that initiated the command. It is the primary mechanism for identifying the source of a command invocation.

## Lifecycle & Ownership
As an interface, CommandOwner itself has no lifecycle. However, the lifecycle of its *implementations* is critical to system stability.

- **Creation:** Implementing objects (e.g., a Player entity) are created by their respective subsystems. A Player is created by the network session manager upon connection; the Console owner is created during server bootstrap.
- **Scope:** The scope of a CommandOwner implementation is tied to the entity it represents. A Player's CommandOwner lives for the duration of their session. The server console's CommandOwner is a singleton that persists for the entire server lifetime.
- **Destruction:** Implementations are destroyed along with the entity they represent. For a Player, this occurs on disconnection. For the server console, this occurs on server shutdown.

**WARNING:** Systems holding a reference to a CommandOwner must be aware of its underlying lifecycle. Holding a reference to a disconnected Player's CommandOwner can lead to memory leaks or invalid state exceptions.

## Internal State & Concurrency
The CommandOwner contract itself is stateless.

- **State:** The interface mandates no state. However, implementations are expected to derive their name from their own internal, and potentially mutable, state. For example, a Player implementation might retrieve the name from its profile data.
- **Thread Safety:** The contract does not enforce thread safety. It is the responsibility of the implementing class to ensure that the getName method is thread-safe, as the CommandSystem may invoke it from various worker threads. Returning a simple, immutable String is the standard and safest implementation.

## API Surface
The public contract is minimal, focused solely on identity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getName() | String | O(1) | Returns the canonical, non-null name of the command issuer. |

**WARNING:** The getName method must **never** return null. Doing so will result in a NullPointerException deep within the command processing pipeline. Implementations should return a sensible default, such as "Unknown", if a name is not available.

## Integration Patterns

### Standard Usage
The primary integration pattern is implementation. A class that can execute commands, such as a Player or Console, must implement this interface.

```java
// Example of a Player class implementing the contract
public class Player implements CommandOwner {
    private final String username;

    public Player(String username) {
        this.username = username;
    }

    @Override
    public String getName() {
        // The name is derived from the Player's core state
        return this.username;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Returning Null:** Never return null from getName. This violates the core contract of identity.
- **Dynamic Name Generation:** Avoid computationally expensive operations within getName. This method may be called frequently for logging and permission checks. The name should be cached or readily available.
- **Mutable Return Value:** While String is immutable in Java, be cautious if returning any other type in a similar contract. The CommandSystem expects a stable identifier for the duration of a command's execution.

## Data Pipeline
The data provided by CommandOwner is used to tag and route command execution flows.

> Flow:
> Raw Command Input -> Command Parser -> **CommandOwner.getName()** (for logging/permissions) -> Command Executor -> Feedback sent to originator identified by CommandOwner
---

