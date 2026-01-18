---
description: Architectural reference for the Options class, the server's command-line argument parser and configuration provider.
---

# Options

**Package:** com.hypixel.hytale.server.core
**Type:** Utility

## Definition
```java
// Signature
public class Options {
```

## Architecture & Concepts
The Options class is the definitive source of truth for all server startup configurations provided via command-line arguments. It serves as the initial configuration layer, translating raw string arguments from the operating system into strongly-typed, queryable Java objects. This class is one of the first to be invoked during the server's bootstrap sequence.

Its core design relies on the third-party library JOpt Simple. Each static `OptionSpec` field, such as `BIND` or `ASSET_DIRECTORY`, defines a valid command-line flag, its expected value type, and any default values. This approach centralizes the entire command-line API of the server into a single, self-documenting file.

Nested `ValueConverter` implementations (e.g., `PathConverter`, `SocketAddressValueConverter`) provide the logic for converting string inputs into complex types, ensuring that by the time other systems query for a configuration value, it is already parsed, validated, and correctly typed. This prevents configuration parsing logic from being scattered throughout the codebase.

## Lifecycle & Ownership
- **Creation:** The internal state of this class, a static `OptionSet` object, is created and populated exclusively by the static `parse(String[] args)` method. This method is designed to be called only once by the server's main entry point at the very beginning of the application lifecycle.
- **Scope:** The parsed options are stored in a static field. Therefore, the configuration state is global and persists for the entire lifetime of the server process (the JVM instance).
- **Destruction:** The state is not explicitly destroyed or garbage collected. It is cleared only when the server process terminates.

## Internal State & Concurrency
- **State:** The primary state is held in the private static `optionSet` field. This object is mutable during the single invocation of the `parse` method. After this initial call, it is treated as effectively immutable by the rest of the application. The public `OptionSpec` fields are immutable definitions.
- **Thread Safety:** This class is **not thread-safe** for writes. The `parse` method modifies global static state and must never be called from multiple threads or more than once during the application's lifetime. All read operations, such as `getOptionSet` or querying values from the returned `OptionSet`, are safe to perform from any thread *after* the initial parsing is complete. This follows a "write-once, read-many" pattern suitable for application configuration.

**Warning:** Any attempt to call `parse` after the initial server bootstrap will lead to unpredictable behavior and race conditions.

## API Surface
The public API consists of the static `OptionSpec` fields which define the configuration contract, and the methods to parse and access the results.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HELP | OptionSpec<Void> | N/A | Defines the `--help` flag. |
| BIND | OptionSpec<InetSocketAddress> | N/A | Defines the `--bind` or `-b` flag for the server's network address. |
| ASSET_DIRECTORY | OptionSpec<Path> | N/A | Defines the `--assets` flag for the primary asset directory path. |
| SINGLEPLAYER | OptionSpec<Void> | N/A | Defines the `--singleplayer` flag to run the server in singleplayer mode. |
| parse(String[] args) | boolean | O(N) | Parses arguments and populates internal state. Returns true if the server should exit immediately (e.g., after printing help). |
| getOptionSet() | OptionSet | O(1) | Retrieves the globally parsed `OptionSet`. Returns null if `parse` has not been called. |
| getOrDefault(spec, set, def) | T | O(1) | A static helper to safely retrieve an option's value or a provided default. |

## Integration Patterns

### Standard Usage
The `parse` method should be called in the main entry point of the application. Other systems then retrieve the static `OptionSet` to configure themselves.

```java
// In the server's main(String[] args) method
public static void main(String[] args) {
    try {
        // Parse arguments. If true, an action like --help was handled and we should exit.
        if (Options.parse(args)) {
            return;
        }
    } catch (IOException e) {
        System.err.println("Failed to parse options: " + e.getMessage());
        System.exit(1);
    }

    // Proceed with server startup...
}

// Elsewhere, for example in a NetworkManager initialization method
public void initialize() {
    OptionSet config = Options.getOptionSet();
    InetSocketAddress bindAddress = config.valueOf(Options.BIND);
    
    // Use the retrieved configuration to bind the server socket
    this.server.bind(bindAddress);
}
```

### Anti-Patterns (Do NOT do this)
- **Re-parsing Arguments:** Do not call `Options.parse(args)` more than once. This will overwrite the global configuration and is not an atomic operation, potentially causing other threads to read inconsistent state.
- **Pre-emptive Access:** Do not call `Options.getOptionSet()` before the main entry point has successfully called `Options.parse()`. This will return null and cause a `NullPointerException` in any system that tries to use it. The application's bootstrap sequence must enforce this order of operations.
- **Direct Instantiation:** This is a utility class with only static members. Do not attempt to create an instance using `new Options()`.

## Data Pipeline
The Options class is the origination point for all command-line configuration data. It does not process a stream of data but rather creates the initial configuration snapshot that feeds into the rest of the server.

> Flow:
> Command Line `String[]` -> **Options.parse()** -> Static `OptionSet` (Internal State) -> Configuration Consumers (Network, AssetManager, Universe, etc.)

