---
description: Architectural reference for SuggestionProvider
---

# SuggestionProvider

**Package:** com.hypixel.hytale.server.core.command.system.suggestion
**Type:** Functional Interface / Strategy

## Definition
```java
// Signature
@FunctionalInterface
public interface SuggestionProvider {
   void suggest(@Nonnull CommandSender var1, @Nonnull String var2, int var3, @Nonnull SuggestionResult var4);
}
```

## Architecture & Concepts
The SuggestionProvider interface defines a behavioral contract for generating command argument suggestions, commonly known as tab-completion. It is a core component of the server's command processing system, implementing the Strategy Pattern.

This design decouples the central command parsing engine from the domain-specific logic required to suggest completions for different argument types. For example, one implementation might suggest online player names, while another suggests available world names or material types. The command system invokes the appropriate SuggestionProvider based on the command and argument being typed by the user, delegating the responsibility of suggestion generation entirely to the provider's implementation.

This interface is the primary extension point for developers to add rich, context-aware tab-completion to custom commands.

## Lifecycle & Ownership
- **Creation:** Implementations of SuggestionProvider, typically as lambda expressions or anonymous classes, are created and registered with a command argument definition. This registration occurs during the command's construction, usually at server startup or when a plugin is loaded. The interface itself is never instantiated.
- **Scope:** An implementation's lifecycle is bound to the command definition it is associated with. It exists as a stateless, function-like object for the duration that the command is registered with the server.
- **Destruction:** The implementation object is eligible for garbage collection when its associated command is unregistered, for instance, during a plugin unload or server shutdown.

## Internal State & Concurrency
- **State:** Implementations of SuggestionProvider are **expected to be stateless**. They should operate exclusively on the provided arguments and the current, safely-accessed game state. Any form of internal caching must be managed with extreme care to prevent memory leaks and stale data.
- **Thread Safety:** The `suggest` method is typically invoked on the main server thread. However, implementations **must be thread-safe and re-entrant**. Do not assume a single-threaded calling context. All access to shared, mutable game state (e.g., a list of online players) must be performed through thread-safe APIs provided by the engine to prevent concurrency violations.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| suggest(sender, input, cursor, result) | void | Implementation-Defined | Populates the `result` object with suggestions based on the current command `input` and `sender` context. This is a high-frequency, performance-critical method. |

## Integration Patterns

### Standard Usage
The intended use is to provide a lambda expression during the construction of a command argument. The system will then automatically invoke this logic when a user requests tab-completion for that argument.

```java
// Example of registering a provider for a command argument
ArgumentBuilder.argument("playerName", StringArgumentType.string())
    .suggests((sender, input, cursor, result) -> {
        // Logic to find online players matching 'input'
        // and add them to the 'result' object.
        // Must be fast and thread-safe.
    })
    .executes(context -> { /* ... */ });
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** Never perform slow or blocking operations within the `suggest` method, such as network requests, database queries, or heavy file I/O. Doing so will block the server's command processing thread and cause severe performance degradation, effectively freezing the user's client.
- **Stateful Implementations:** Avoid storing per-user or per-invocation state as instance fields on your provider implementation. This is a common source of memory leaks and concurrency bugs, as a single provider instance is shared across all invocations for that command argument.
- **Ignoring Context:** Failing to use the provided `CommandSender` to filter suggestions is a security and usability flaw. Always filter results based on the sender's permissions and game context to avoid leaking information or suggesting invalid options.

## Data Pipeline
The flow of data for a tab-completion request demonstrates the role of the SuggestionProvider.

> Flow:
> Client Input (`/command partial_arg<TAB>`) -> Network Packet -> Server Command Parser -> **SuggestionProvider.suggest()** -> SuggestionResult -> Network Response -> Client UI Renders Suggestions

