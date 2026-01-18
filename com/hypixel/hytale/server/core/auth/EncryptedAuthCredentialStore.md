---
description: Architectural reference for EncryptedAuthCredentialStore
---

# EncryptedAuthCredentialStore

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Transient

## Definition
```java
// Signature
public class EncryptedAuthCredentialStore implements IAuthCredentialStore {
```

## Architecture & Concepts

The EncryptedAuthCredentialStore is a concrete implementation of the IAuthCredentialStore interface, providing a secure, disk-backed persistence layer for server authentication credentials. Its primary responsibility is to store and retrieve OAuth 2.0 tokens and the associated user profile UUID, ensuring this sensitive data can survive server restarts.

The core architectural principle is **machine-locked security**. The encryption key is derived at runtime using a PBKDF2 function that combines a static, hardcoded salt with a unique hardware identifier retrieved from the host machine via HardwareUtil. This design intentionally tethers the encrypted credentials file to the specific hardware it was created on. An attempt to copy the file to another machine and decrypt it will fail, as the derived key will not match.

Data is serialized into the BSON format using a statically defined BuilderCodec before being encrypted with AES/GCM. This provides both a structured data format and authenticated encryption, protecting against tampering. The class acts as a low-level utility, intended to be managed by a higher-level authentication service that orchestrates the login flow.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly by a higher-level service, such as a server's main entry point or an authentication manager. The constructor requires a file Path, which dictates the storage location. Upon instantiation, it immediately attempts to derive the encryption key and load any pre-existing credentials from the specified path.

-   **Scope:** The object's lifetime is bound to its owner. It is designed to persist for the entire duration of a server session to provide a consistent view of the stored credentials.

-   **Destruction:** The object is eligible for garbage collection when its owner is de-referenced. There is no explicit shutdown method. The clear method provides a way to logically reset the store and physically delete the credentials file from the disk.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains the current OAuthTokens and profile UUID in memory. A critical piece of its initial state is the encryptionKey, which is derived once in the constructor. If key derivation fails, the instance operates in a non-persistent mode where all data is lost when the object is destroyed. The in-memory state is intended to be a cache of the on-disk, encrypted state.

-   **Thread Safety:** This class is **not thread-safe**. All public methods that modify state, such as setTokens and setProfile, perform non-atomic read-modify-write operations on shared fields. These methods also involve blocking file I/O. Concurrent access from multiple threads without external synchronization will lead to race conditions and potential file corruption. All interactions with an instance of this class must be synchronized externally or confined to a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setTokens(OAuthTokens tokens) | void | O(N) | Persists the given tokens to disk. This is a blocking I/O operation. |
| getTokens() | OAuthTokens | O(1) | Returns the current in-memory tokens. Does not read from disk. |
| setProfile(UUID uuid) | void | O(N) | Persists the given profile UUID to disk. This is a blocking I/O operation. |
| getProfile() | UUID | O(1) | Returns the current in-memory profile UUID. |
| clear() | void | O(N) | Clears credentials from memory and deletes the backing file from disk. |

## Integration Patterns

### Standard Usage
The EncryptedAuthCredentialStore should be instantiated once per logical credential set (typically once per server instance) and managed by a central authentication service.

```java
// In a server initialization routine
Path credentialPath = Paths.get("./server_credentials.bin");
IAuthCredentialStore credentialStore = new EncryptedAuthCredentialStore(credentialPath);

// On startup, check for existing tokens to re-authenticate
OAuthTokens existingTokens = credentialStore.getTokens();
if (existingTokens.isValid()) {
    // Attempt to refresh or use the existing session
}

// After a new successful login, save the new tokens
OAuthTokens newTokens = ... // from login service
credentialStore.setTokens(newTokens);
```

### Anti-Patterns (Do NOT do this)
-   **Multiple Instances for a Single File:** Do not create more than one EncryptedAuthCredentialStore instance pointing to the same file path. This will create independent in-memory caches, leading to severe race conditions and data loss when they attempt to write to the file.

-   **Concurrent Modification:** Never call setTokens, setProfile, or clear from multiple threads without external locking. The internal state will become inconsistent with the persisted file.

-   **Ignoring Key Derivation Failure:** The constructor proceeds even if the hardware-based encryption key cannot be derived. In this state, the store acts as a volatile, in-memory-only cache. Relying on it for persistence without checking for the initial warning logs can lead to unexpected data loss on server restart.

## Data Pipeline

The flow of data for persistence (saving) and retrieval (loading) is critical to its operation.

> **Save Flow:**
> In-memory OAuthTokens -> Internal StoredCredentials DTO -> BSON Serialization -> Plaintext byte array -> AES/GCM Encryption -> Filesystem Write

> **Load Flow:**
> Filesystem Read -> Encrypted byte array -> AES/GCM Decryption -> Plaintext byte array -> BSON Deserialization -> Internal StoredCredentials DTO -> In-memory OAuthTokens

