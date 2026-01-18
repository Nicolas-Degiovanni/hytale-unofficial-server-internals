---
description: Architectural reference for AssetsDuplicatesCommand
---

# AssetsDuplicatesCommand

**Package:** com.hypixel.hytale.server.core.command.commands.debug
**Type:** Transient

## Definition
```java
// Signature
public class AssetsDuplicatesCommand extends AbstractAsyncCommand {
```

## Architecture & Concepts
The AssetsDuplicatesCommand is a server-side diagnostic tool designed to identify redundant asset data and report the potential disk space savings. It operates within the server's Command System framework and is intended for use by developers or server administrators to audit game and mod assets.

Architecturally, this command's most significant feature is its asynchronous execution model, inherited from AbstractAsyncCommand. This design is critical because the command's primary function involves I/O operations—calculating the file sizes of numerous assets. By performing these operations off the main server thread, it prevents the game loop from stalling, ensuring server performance and responsiveness are not impacted during the scan.

The command acts as a read-only client to the CommonAssetRegistry, which serves as the single source of truth for all loaded asset metadata. It queries the registry for assets that share identical content hashes and then processes this information to generate a user-friendly report.

## Lifecycle & Ownership
- **Creation:** A prototype instance of AssetsDuplicatesCommand is instantiated by the Command System during server bootstrap when all commands are discovered and registered. It is not created on-demand per execution.
- **Scope:** The prototype instance is a singleton that persists for the entire lifetime of the server. However, the state and resources associated with a specific execution (the CompletableFuture chain, the list of duplicates, etc.) are transient and scoped exclusively to a single invocation of the `executeAsync` method.
- **Destruction:** The prototype instance is discarded and garbage collected during server shutdown. All execution-specific objects are eligible for garbage collection as soon as the command completes and the final message is delivered to the user.

## Internal State & Concurrency
- **State:** The class itself holds minimal, immutable state in the form of its command definition and the reverseFlag argument parser. All mutable state required for execution, such as the list of duplicated assets and calculated sizes, is confined to the local method scope of `executeAsync`. This design prevents state leakage between command invocations.

- **Thread Safety:** This class is **not thread-safe** for concurrent execution on a single instance. However, the server's Command System is designed to invoke a command's execution logic serially, mitigating this risk. The internal logic is concurrency-aware; it safely orchestrates multiple parallel I/O operations using a `CompletableFuture` fan-out/fan-in pattern (`CompletableFuture.allOf`). This ensures that file size calculations happen concurrently on a worker thread pool without data races, and the final aggregation step only occurs after all parallel tasks have completed.

## API Surface
The primary contract is defined by its parent, AbstractAsyncCommand.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| executeAsync(context) | CompletableFuture<Void> | O(N log N) | Executes the asset duplication scan. The complexity is dominated by I/O latency for N duplicated assets and the subsequent sorting. This method is non-blocking and returns a future that completes when the report has been sent to the user. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. It is designed to be invoked by a user (e.g., a server administrator) through the server console or in-game chat. The Command System handles parsing, routing, and execution.

*Conceptual invocation by the server console:*
```
/assets duplicates --reverse
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance via `new AssetsDuplicatesCommand()`. The Command System manages the lifecycle of command objects. Direct creation bypasses registration and context injection.
- **Blocking on the Future:** Calling `.get()` or `.join()` on the `CompletableFuture` returned by `executeAsync` from a main engine thread is a severe anti-pattern. This will block the thread and negate all benefits of the asynchronous design, potentially freezing the server.
- **Stateful Modification:** Do not attempt to add mutable instance fields to this class. The command instance may be reused, and shared state would lead to race conditions and incorrect behavior.

## Data Pipeline
The command orchestrates a data flow that begins with user input and ends with a formatted report, performing significant I/O and data transformation in between.

> Flow:
> User Input (`/assets duplicates`) -> Command System Parser -> **AssetsDuplicatesCommand.executeAsync** -> Query CommonAssetRegistry -> Parallel I/O for File Sizes -> Aggregate & Sort Results -> Format `Message` Objects -> CommandContext.sendMessage -> User Console

