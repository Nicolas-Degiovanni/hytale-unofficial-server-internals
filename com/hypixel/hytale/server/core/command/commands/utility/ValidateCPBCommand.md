---
description: Architectural reference for ValidateCPBCommand
---

# ValidateCPBCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility
**Type:** Transient

## Definition
```java
// Signature
public class ValidateCPBCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The **ValidateCPBCommand** is a server-side diagnostic utility that integrates into the Hytale command system. Its primary architectural purpose is to provide a mechanism for validating the integrity of prefab asset files (`.prefab.json`) without halting server execution.

By extending **AbstractAsyncCommand**, this command delegates its core logic to a background thread pool managed by the `CompletableFuture` framework. This is a critical design choice, as traversing file systems and deserializing potentially thousands of asset files is a highly I/O-bound and CPU-intensive operation that would otherwise cause significant server lag or unresponsiveness.

The command acts as an orchestrator, interacting with several core server systems:
*   **Command System:** It registers the `validatecpb` command, parses arguments like the optional file path, and receives a **CommandContext** for execution.
*   **AssetModule:** It queries this module to discover all registered asset packs when no specific path is provided, enabling a full-system validation.
*   **BsonPrefabBufferDeserializer:** This is the workhorse component that performs the actual deserialization and validation of the BSON-like prefab data. **ValidateCPBCommand** is responsible for feeding file paths to this deserializer and handling its success or failure outcomes.

## Lifecycle & Ownership
- **Creation:** A single instance of **ValidateCPBCommand** is instantiated by the server's command registration system during the server bootstrap sequence. It is registered under the name `validatecpb`.
- **Scope:** The command object instance persists for the entire lifetime of the server session. However, its execution context is transient and created anew for each command invocation.
- **Destruction:** The object is dereferenced and becomes eligible for garbage collection when the server shuts down and the command registry is cleared.

## Internal State & Concurrency
- **State:** This class is effectively stateless. Its only field, **pathArg**, is a definition for a command argument and is immutable after the constructor is called. All state related to an execution, such as the list of failed prefabs, is confined to method-local scope within the `convertPrefabs` static method.

- **Thread Safety:** The command instance itself is thread-safe due to its stateless nature. However, the implementation of its core logic contains a **critical concurrency flaw**.

    **WARNING:** The `convertPrefabs` method collects failure messages from multiple parallel `CompletableFuture` tasks into a standard `it.unimi.dsi.fastutil.objects.ObjectArrayList`. This collection is **not thread-safe**. Concurrent calls to `failed.add` from multiple background threads will result in a race condition, potentially leading to lost error messages or internal corruption of the list. For a robust implementation, a concurrent collection such as `java.util.concurrent.ConcurrentLinkedQueue` should be used.

## API Surface
The public contract is defined by its nature as a command and its inheritance from **AbstractAsyncCommand**. Direct invocation of its methods is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(N * M) | Asynchronously executes the prefab validation. N is the number of prefabs; M is the complexity of deserializing a single prefab. This method orchestrates the file scan and deserialization on a background thread. |

## Integration Patterns

### Standard Usage
This command is intended to be executed by a server administrator or developer via the server console to diagnose asset issues.

```java
// Example 1: Validate all prefabs in all loaded asset packs
validatecpb

// Example 2: Validate all prefabs within a specific directory
validatecpb my_assets/prefabs/
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ValidateCPBCommand()`. The command system manages the lifecycle of command objects. Direct instantiation will result in an object that is not registered with the server.
- **Direct Invocation:** Do not call the **executeAsync** method directly. This bypasses the command system's parsing, permission checks, and context provisioning.
- **Ignoring Concurrency Flaw:** Do not rely on the failure report from this command in a production environment without first addressing the race condition in the `convertPrefabs` method. The list of failed files may be incomplete.

## Data Pipeline
The data flow for this command is initiated by user input and terminates with feedback to that same user. The primary transformation is from file paths to deserialized in-memory objects, with the process being used for validation rather than state change.

> Flow:
> Console Input -> Command System Parser -> **ValidateCPBCommand::executeAsync** -> Files.walk Stream -> BsonUtil::readDocument -> BsonPrefabBufferDeserializer::deserialize -> CommandContext::sendMessage -> Console Output

