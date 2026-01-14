## 🧩ID & Session System

### UID (Unique Player ID)
- Generated **server-side only** using **Snowflake**
- 64-bit `long` (must read via `ReadInt64()` client-side)
- Structure: `[timestamp][machineId][sequence]`
- MachineId assigned **per server instance** (not client)
- Timestamp uses custom Epoch to reduce bits

### SessionID (Token)
- Created **after login success**
- Bound to `UID` + `connection`
- Returned to client in login response
- Used to validate **every** subsequent request incl. heartbeat
- **Never** stored on disk by client

---

## 🔐Server: SessionManager
Stores:
- `Uid`
- `SessionId`
- `ExpireTime` (refresh on heartbeat)
- `Connection` reference

Uses:
- `lock` for thread-safe access due to multi-threading
- `Validate()` for every incoming packet
- `TimeoutChecker` running via `Task.Run()` (ThreadPool)

Timeout logic:
- Heartbeat every `5s`
- Expire after `15–25s`
- On expire → disconnect + remove session

---

## 🌐Client Architecture

### Separation of concerns
| Layer | Responsibility |
|------|----------------|
| ClientSession | Pure networking (TCP send/recv) |
| ClientSessionStore | Stores UID & SessionID in memory |
| GameClient | Handles heartbeat + gameplay messages |

### ClientSessionStore
```csharp
public static class ClientSessionStore {
    public static long Uid { get; private set; }
    public static string SessionId { get; private set; }
    public static bool IsLoggedIn => Uid > 0 && !string.IsNullOrEmpty(SessionId);
    public static void Set(long uid, string sessionId) =>
        (Uid, SessionId) = (uid, sessionId);
}
```

---

## 📡Heartbeat
Sent **after login only**:

```json
{
  "type": "heartbeat",
  "uid": 289101829182910,
  "sessionId": "7GnzQPxxLH2j3Xk9",
  "timestamp": 1733655321123
}
```

Server:
- validate session
- refresh expiration
- update online status

---

## 🕒Timestamp
Use **Unix millisecond timestamp**:

```csharp
long timestamp = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
```

Avoid `DateTime.Now` (timezone issues).

---

## ✔️Key Rules

| Feature | Client | Server |
|--------|--------|--------|
| UID generation | ❌ | ✔️ Snowflake |
| SessionID generation | ❌ | ✔️ On login |
| Store UID | ✔️ memory | ✔️ memory/db |
| Store SessionID | ✔️ memory | ✔️ memory/Redis |
| Validate messages | ❌ | ✔️ UID+SessionID |
| Timeout management | ❌ | ✔️ best on server |
| Heartbeat scheduling | ✔️ | ✔️ verify only |

---

## 🏁 Final Architecture Flow

```
TCP Connect
       ↓
ClientSession created (no identity)
       ↓
Login Request
       ↓
AUTH OK → Generate (UID + SessionId)
Store in SessionManager
       ↓
Return UID+SessionId to client
       ↓
Heartbeat every 5s + Validate + Refresh ExpireTime
       ↓
Timeout or Logout → Session removed, connection closed
```

---
