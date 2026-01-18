---
description: Architectural reference for EncryptedAuthCredentialStoreProvider
---

# EncryptedAuthCredentialStoreProvider

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Transient Factory

## Definition
```java
// Signature
public class EncryptedAuthCredentialStoreProvider implements AuthCredentialStoreProvider {
```

## Architecture & Concepts
The EncryptedAuthCredentialStoreProvider is a factory component responsible for instantiating an `EncryptedAuthCredentialStore`. It acts as a bridge between the server's static configuration and the runtime creation of the credential storage mechanism.

This class is a concrete implementation of the **Strategy Pattern**, where the server can be configured to use different types of credential stores (e.g., encrypted, plaintext, in-memory). This provider represents the "Encrypted" strategy.

Its primary architectural feature is the static `CODEC` field. This exposes a serialization and deserialization contract, allowing the Hytale configuration system to dynamically construct and configure an instance of this provider from a data source, such as a JSON or YAML file. The provider's behavior, specifically the file path for the encrypted store, is determined entirely by the data supplied to this codec during server initialization.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via the constructor. They are materialized by the server's configuration loader, which uses the public static `CODEC` to deserialize the provider from a configuration file. This process populates the internal `path` field.
- **Scope:** The object is short-lived. It exists only during the server's bootstrap sequence. Its purpose is fulfilled once the `createStore` method is called and the resulting `IAuthCredentialStore` is registered with the server's authentication service.
- **Destruction:** The provider instance is eligible for garbage collection immediately after the bootstrap phase completes. It holds no persistent references and is not retained by any long-lived services.

## Internal State & Concurrency
- **State:** The class holds a single mutable state field: `path`. This field is populated once during deserialization and is not expected to change thereafter. It defaults to `auth.enc` if not specified in the configuration.
- **Thread Safety:** This class is **not thread-safe**. The `path` field is not protected against concurrent modification. However, concurrency is not a concern under the intended usage pattern: the object is created, configured, and used in a single-threaded context during server startup. The `createStore` method itself is re-entrant and safe, as it only reads the internal state to create a new object.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ID | static String | O(1) | The unique identifier ("Encrypted") used in configuration files to select this provider. |
| CODEC | static BuilderCodec | O(1) | The serialization contract used by the engine to construct this object from configuration data. |
| createStore() | IAuthCredentialStore | O(1) | Factory method. Instantiates and returns a new EncryptedAuthCredentialStore using the configured path. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is configured declaratively within the server's configuration files. The system uses the `ID` to identify and the `CODEC` to instantiate the provider.

A typical configuration snippet might look like this:
```yaml
# Example server_config.yaml
auth:
  store:
    type: "Encrypted"
    path: "credentials/server_auth.bin"
```
The server's configuration loader would parse this, find the provider with the ID "Encrypted", and use its `CODEC` to create an `EncryptedAuthCredentialStoreProvider` with its `path` field set to "credentials/server_auth.bin".

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new EncryptedAuthCredentialStoreProvider()`. This bypasses the configuration system and hardcodes the default path, making the server less flexible and potentially causing conflicts if a path is also specified in a configuration file.
- **Post-Creation State Mutation:** Do not modify the `path` field after the provider has been instantiated by the configuration system. The object's state should be considered immutable after its initial creation.

## Data Pipeline
This class is not part of a runtime data processing pipeline. It is a factory component used during the server's initialization phase. Its role is to translate configuration data into a live service object.

> Flow:
> Server Configuration File -> Hytale Codec Engine -> **EncryptedAuthCredentialStoreProvider Instance** -> `createStore()` call -> `IAuthCredentialStore` Service Registration

