---
description: Architectural reference for DefaultAuthCredentialStore
---

# DefaultAuthCredentialStore

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Transient

## Definition
```java
// Signature
public class DefaultAuthCredentialStore implements IAuthCredentialStore {
```

## Architecture & Concepts
The DefaultAuthCredentialStore is a fundamental, in-memory data container responsible for holding authentication state for a single client session on the server. It acts as a transient, session-scoped cache for OAuth tokens and the associated user profile UUID.

This class does not perform authentication itself. Instead, it serves as the canonical storage for the *results* of an authentication process. Higher-level services, such as an AuthenticationManager, populate this store after successfully validating a client. Subsequently, other server systems query this store to confirm a session's identity without repeatedly accessing external authentication providers. Its primary role is to decouple the authentication state from the network connection lifecycle, providing a clean interface for identity-aware services.

## Lifecycle & Ownership
The lifecycle of a DefaultAuthCredentialStore instance is tightly bound to a single user's network session. Mismanagement of its lifecycle can lead to severe security vulnerabilities.

- **Creation:** An instance is typically created by a session management service (e.g., ConnectionManager) at the beginning of the authentication handshake for a new, incoming client connection.
- **Scope:** The object is designed to live for the exact duration of an authenticated client session. It should be uniquely associated with one and only one client connection at any given time.
- **Destruction:** The instance becomes eligible for garbage collection when the client's session is terminated, either through a clean disconnect or a timeout. The clear method **must** be invoked as part of the session teardown sequence to prevent credential leakage if the object were to be pooled or reused, although pooling is strongly discouraged.

## Internal State & Concurrency
- **State:** This component is highly mutable and stateful by design. It contains the OAuthTokens and profile UUID, which are written once upon successful authentication and read many times throughout the session.
- **Thread Safety:** **CRITICAL WARNING:** This class is **not thread-safe**. It contains no internal synchronization mechanisms (e.g., locks, volatile fields). All access to an instance of this class **must** be externally synchronized or confined to a single thread, such as a dedicated network event loop or a per-session actor. Concurrent writes from multiple threads will lead to race conditions, memory visibility issues, and unpredictable authentication state.

## API Surface
The public contract is minimal, focused exclusively on state management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setTokens(OAuthTokens tokens) | void | O(1) | Overwrites the current OAuth tokens. Not null-safe. |
| getTokens() | OAuthTokens | O(1) | Retrieves the currently stored tokens. |
| setProfile(UUID uuid) | void | O(1) | Sets the authenticated user's profile UUID. |
| getProfile() | UUID | O(1) | Retrieves the user's profile UUID. Returns null if not set. |
| clear() | void | O(1) | Resets all internal state to its initial, unauthenticated condition. |

## Integration Patterns

### Standard Usage
The store should be managed by a session context or service registry. It is populated by the authentication system and read by other game logic services.

```java
// During authentication success
IAuthCredentialStore credStore = session.getCredentialStore();
credStore.setTokens(retrievedTokens);
credStore.setProfile(userProfileUUID);

// During a subsequent game logic request
IAuthCredentialStore credStore = session.getCredentialStore();
UUID playerId = credStore.getProfile();
if (playerId != null) {
    // Proceed with player-specific logic
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Instances:** Never share a single DefaultAuthCredentialStore instance across multiple user sessions. This will cause catastrophic credential overwrites and security breaches.
- **Unsynchronized Access:** Do not read or write to this object from multiple threads without external locking. This will lead to data corruption.
- **State Reuse:** Do not reuse an instance for a new session without first calling clear. Failure to do so will leak the previous user's session data to the new user.

## Data Pipeline
This component acts as a sink for the authentication process and a source for identity-aware services.

> Flow:
> External OAuth Provider -> Authentication Service -> **DefaultAuthCredentialStore.setTokens()** -> **DefaultAuthCredentialStore.setProfile()**
>
> Game Logic Service -> **DefaultAuthCredentialStore.getProfile()** -> Player Data Lookup

---

