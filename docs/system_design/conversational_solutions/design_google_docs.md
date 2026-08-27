# Design Google Docs (Real-time Collaborative Editing)

## ⏱️ 1. The 2-Minute Version

**Goal**: Design a real-time collaborative document editing service like Google Docs or Etherpad that allows multiple users to edit the same document simultaneously with low latency and eventual consistency.

**Key Components**:
1.  **Collaboration Server**: Central authority for processing edits and broadcasting changes.
2.  **Operational Transformation (OT)**: The core algorithm to resolve concurrent editing conflicts (used by Google Docs). Alternatively, CRDTs.
3.  **WebSocket Server**: Maintains persistent connections for sub-100ms updates.
4.  **Database**: NoSQL (e.g., HBase/Cassandra) for storing operation logs; Document DB or Blob storage for snapshots.

**Key Challenges**:
-   **Concurrency**: User A and User B type at the same time. How do we merge changes so the document looks the same for both?
-   **Latency**: Typing must feel instant. Network delay shouldn't block the UI.
-   **Ordering**: Operations must be applied in the correct order to preserve intent.

**Trade-offs**:
-   **OT vs CRDT**: OT (Centralized, complex logic, data efficient) vs CRDT (Decentralized, simpler merging, memory heavy). Google Docs uses OT.
-   **TCP vs UDP**: WebSocket (TCP) ensures reliable delivery of operations, which is critical for document integrity, even if slightly slower than UDP.

---

## 🏗️ 2. The 10-Minute Structured Version

### Requirements

#### Functional
-   **Real-time Editing**: Multiple users editing simultaneously. Changes appear instantly.
-   **Collaborator Presence**: See who else is viewing/editing and their cursor position.
-   **History/Versioning**: View and restore past versions.
-   **Offline Support**: Edit offline and sync when back online.

#### Non-Functional
-   **Latency**: End-to-end latency < 100ms for a "real-time" feel.
-   **Consistency**: All users must eventually see the exact same document.
-   **Availability**: High availability for document access.
-   **Scalability**: Support 10M concurrent users, 100 concurrent users per document.

### Capacity Estimation
-   **DAU**: 100 Million.
-   **Writes**: Heavy write load. Assume 10% of users are editing at any moment.
-   **Storage**: 
    -   Average document: 100KB (text only).
    -   1 billion docs * 100KB = 100 TB (tiny compared to video, but operation logs grow large).
-   **Bandwidth**: Tiny text operations (JSON), mostly negligible. Connection management is the bottleneck.

### High-Level Architecture

```mermaid
graph TB
    ClientA[User A] <-->|WebSocket| LB[Load Balancer]
    ClientB[User B] <-->|WebSocket| LB
    
    LB --> Gateway[API Gateway / WebSocket Handler]
    
    Gateway <--> SessionService[Collaboration/Session Service]
    
    SessionService --> Redis[Redis (Active Document State)]
    SessionService --> Kafka[Kafka (Operation Stream)]
    
    Kafka --> PersistWorker[Persistence Worker]
    PersistWorker --> DB[(NoSQL - Operation Log)]
    PersistWorker --> ObjStore[S3 (Snapshots)]
```

### Data Flow: The Edit Path
1.  **User A types "Hello"**: Browser applies change locally immediately (optimistic UI).
2.  **Send Operation**: Client sends operation `insert('Hello', index=0)` via WebSocket to **Session Service**.
3.  **OT Processing**:
    -   Session Service looks up current state/version in **Redis**.
    -   If User B also typed, Service transforms User A's operation against User B's.
4.  **Broadcast**: Service sends transformed operation to all other users in the session (User B).
5.  **Persistence**: Operation is queued to **Kafka** to be saved to DB asynchronously.
6.  **Ack**: Server sends acknowledgement to User A.

---

## 🧠 3. Deep Dive & Technical Details

### 1. Operational Transformation (OT)
This is the heart of Google Docs. It solves the concurrency problem.

**The Problem**:
-   Doc: "ABC"
-   User A: `insert("X", pos=0)` -> "XABC"
-   User B: `delete(pos=2)` -> "AB" (deletes 'C')
-   System receives both. If applied naively:
    -   A's op on B's state: `insert("X", 0)` on "AB" -> "XAB" (Correct)
    -   B's op on A's state: `delete(2)` on "XABC" -> "XAC" (WRONG! Deleted 'B' instead of 'C' because indices shifted).

**The Solution (OT)**:
We must **transform** User B's operation based on User A's operation before applying it.
-   `transform(opB, opA)`: Since A inserted 1 char at 0, and B wanted to delete at 2, B's target shifts by +1.
-   New opB': `delete(pos=3)`.
-   Result on "XABC": `delete(3)` -> "XAB". **Consistent!**

**Client-Server OT**:
Google Docs uses a central server as the source of truth.
-   **Revision Numbers**: Every state has a revision number.
-   **History Buffer**: Clients keep a track of pending operations (sent but not acked) and buffer operations (happened while offline/disconnected).

### 2. CRDTs (Conflict-free Replicated Data Types)
*Alternative Approach.*
-   Used by: Trello, Figma (for some parts), Riak.
-   Concept: Data structures that can always be merged without conflict.
-   **How**: Assign a unique, immutable ID to every character (e.g., `0.1`, `0.2`).
-   **Pros**: Decentralized (P2P friendly), works well offline.
-   **Cons**: "Tombstones" (deleted items take space), overhead of storing IDs for every char. OT is generally preferred for formatted text where intent matters most.

### 3. Data Storage Schema
We don't just store the text. We store the **Log of Operations**.

**Operations Table (Cassandra/DynamoDB)**
| DocumentID | RevisionID | UserID | OperationJSON | Timestamp |
| :--- | :--- | :--- | :--- | :--- |
| doc_123 | 100 | user_A | `{op: "ins", ch: "A", pos: 5}` | 10:00:01 |
| doc_123 | 101 | user_B | `{op: "del", len: 1, pos: 5}` | 10:00:02 |

**Snapshot Storage**
Replaying 10,000 operations to load a doc is slow.
-   Every ~100 operations, create a snapshot of the full text.
-   Store in S3 or Object Store.
-   **Read Path**: Load latest Snapshot + Replay subsequent Operations.

### 4. WebSocket & Session Management
-   **Stateful Services**: Designating a specific server to handle a specific document is easier for OT (in-memory state).
-   **Consistent Hashing**: Route all users for `doc_123` to `Server_Node_5`.
-   **Redis Pub/Sub**: If using stateless servers, use Redis to broadcast messages between nodes, but this adds latency. Sticky sessions + Stateful servers are preferred for this specific use case.

### 5. Presence & Cursors
-   Cursor moves are high volume, low criticality.
-   Don't store permanently.
-   Broadcast via Redis Pub/Sub to other users in the session.
-   "User A is at index 15".

---

## ⚖️ 4. Trade-offs & Alternatives

1.  **OT vs CRDT**:
    -   **OT**: Standard for text. Complex implementation. Centralized server helpful.
    -   **CRDT**: Good for distributed systems or weak connections. Memory overhead.

2.  **UDP vs TCP**:
    -   Gaming uses UDP. Text editing uses TCP/WebSockets because dropping a packet means the document is corrupted.

3.  **Poll vs Push**:
    -   Long-polling is "okay" but adds overhead. WebSockets are the industry standard for this.

## 🔗 References
-   [Google Wave Operational Transformation](https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transformation/operational-transformation.html)
-   [Figma's Multiplayer Technology (CRDT-like)](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/)
