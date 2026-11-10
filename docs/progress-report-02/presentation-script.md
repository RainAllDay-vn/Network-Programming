# Presentation Script: Progress Report 02 – System & Protocol Design Review

**Course:** IT4062E – Network Programming  
**Project:** Building a Collaborative File Sharing Application with Access Control  
**Team Members:** Cao Van Dong, Luong Ngoc Vu Long, Tran Le Minh  
**Target Duration:** ~10–12 minutes (~30–40 seconds per slide)  
**Slide Reference:** `docs/progress-report-02/Slide_HUST.pdf` (21 Slides)

---

## Presentation Structure & Speaker Assignment Suggestion

| Speaker | Section | Slides | Estimated Time |
| :--- | :--- | :---: | :---: |
| **Presenter 1** (e.g., Cao Van Dong) | **Part I: Introduction, Scope & Architecture** | Slides 1 – 6 | ~3.5 mins |
| **Presenter 2** (e.g., Luong Ngoc Vu Long) | **Part II: Protocol Framing, Envelopes & Command Catalog** | Slides 7 – 12 | ~3.5 mins |
| **Presenter 3** (e.g., Tran Le Minh) | **Part III: State Machines, Concurrency & Interaction Sequences** | Slides 13 – 21 | ~4.5 mins |

---

## Slide-by-Slide Script

---

### Slide 1: Title Page

- **Visual:** HUST Blue title slide with project title, course code (IT4062E), and team members' names.
- **Duration:** ~25 seconds
- **Speaker:** Presenter 1

> **[Spoken Script]**  
> *"Good morning, Professor and classmates. Welcome to our second progress report presentation for Network Programming, course IT4062E.  
> Our project is **'Building a Collaborative File Sharing Application'** with group-centric workspaces and role-based access control.  
> Our team consists of three members: Cao Van Dong, Luong Ngoc Vu Long, and Tran Le Minh.  
> Today, we are excited to walk you through our comprehensive system architecture and formal protocol design review."*

> **[Transition]**  
> *"Let us begin with the problem statement and the motivation driving our application."*

---

### Slide 2: 1. Application Introduction & Problem Statement

- **Visual:** Two columns — Left: Motivation & Collaboration Needs; Right: Proposed System Solution.
- **Duration:** ~35 seconds
- **Speaker:** Presenter 1

> **[Spoken Script]**  
> *"In modern collaborative environments—such as student project groups, engineering teams, and research labs—members constantly share documents, codebases, and large datasets.  
> However, unmanaged or ad-hoc shared directories suffer from three major vulnerabilities:  
> 1. **Accidental data loss**: When any member has unrestricted delete or rename permissions, critical project files can be inadvertently deleted or overwritten.  
> 2. **Network fragility**: Real-world TCP connections over the internet can drop mid-stream, leaving corrupt, half-written 'ghost files' on storage without transactional controls.  
> 3. **Concurrency collisions**: Unsynchronized simultaneous access can cause race conditions.  
>  
> To solve this, we propose a centralized, group-isolated file sharing system. Each team has a dedicated workspace with nested subfolders. We enforce a rigid Role-Based Access Control model where any member can contribute and read files, but only designated Group Admins have the authority to delete or rename them. Furthermore, we designed an atomic commit staging protocol over TCP to guarantee that files are never corrupted by network disconnections."*

> **[Transition]**  
> *"Next, let's examine the system actors and core use cases."*

---

### Slide 3: 2. System Functionality & Use Case Model

- **Visual:** Left: System Actors & Scope (Guest, Regular Member, Group Admin); Right: Core Collaboration Use Cases (UC1 to UC9).
- **Duration:** ~35 seconds
- **Speaker:** Presenter 1

> **[Spoken Script]**  
> *"Our system defines three distinct actors:  
> - **Guest**: An unauthenticated user who can register an account or log in with credentials.  
> - **Regular Member**: A user who can create new groups, submit join requests to existing groups, browse directory trees, upload files, download shared files, and create subdirectories.  
> - **Group Admin**: The creator and owner of the group, who holds exclusive governance over member admissions, folder renames, and file deletions.  
>  
> These actors interact through nine core use cases: from authentication and group discovery to hierarchical directory navigation, chunked file transfers with SHA-256 integrity verification, admin safety controls, and real-time push event notifications whenever a group's content changes."*

> **[Transition]**  
> *"To ensure security throughout these use cases, we developed a formal RBAC specification."*

---

### Slide 4: 3. Role-Based Access Control (RBAC) Specification

- **Visual:** Top block defining group-scoped policy; central table showing permissions for 9 operations across Guest, Non-Member, Member, and Group Admin.
- **Duration:** ~40 seconds
- **Speaker:** Presenter 1

> **[Spoken Script]**  
> *"Here is our Role-Based Access Control matrix. Security in our system is strictly **group-scoped**, meaning a user's permissions are dynamically validated on every single incoming command based on the tuple: `(userId, groupId, role)`.  
>  
> Notice the strict separation of privileges:  
> - Public operations like login and registration are open to guests.  
> - Any authenticated user can create a group, immediately becoming its Group Admin.  
> - Within a group, approved regular members enjoy full read and write access—they can list directories, download files, upload new files, and create nested subfolders.  
> - However, high-risk administrative operations—specifically **`FILE_RENAME`**, **`DIR_RENAME`**, **`FILE_DELETE`**, and **`DIR_DELETE`**—are strictly reserved for the Group Admin.  
> If a regular member attempts to rename or delete an asset, the server-side security interceptor immediately intercepts the request and rejects it with a `403 FORBIDDEN` response without touching the file system."*

> **[Transition]**  
> *"Now, let us discuss how this functional model translates into our system architecture."*

---

### Slide 5: 4. System Architecture: Paradigm Evaluation

- **Visual:** Left: Evaluation comparing Pure P2P, Hybrid, and Client-Server; Right: TikZ System Topology diagram.
- **Duration:** ~40 seconds
- **Speaker:** Presenter 1

> **[Spoken Script]**  
> *"Before settling on an architecture, we evaluated three major networking paradigms:  
> - **Pure Peer-to-Peer (P2P)** was discarded because peer connections require complex NAT and firewall traversal like STUN and TURN. More critically, files become unavailable the moment a peer goes offline, and decentralized consensus for RBAC roles is overly complex.  
> - A **Hybrid Architecture**—using a central tracker for indexing and P2P for data transfer—was also rejected due to high signaling overhead and unpredictable peer upload speeds.  
> - We chose the **Client-Server Architecture**. It provides three decisive advantages:  
>   1. **Authoritative Access Control**: The server serves as the single source of truth for user authentication and role verification.  
>   2. **24/7 File Availability**: Group files remain persistently accessible on disk regardless of individual member online status.  
>   3. **Centralized Concurrency Management**: The server coordinates locking and transactional staging, preventing race conditions.  
>  
> As shown in our topology diagram, multiple desktop clients establish persistent TCP connections to our central server engine, which coordinates access to an SQLite metadata database and the partitioned group file repository."*

> **[Transition]**  
> *"Let's zoom into the internal engine of the server."*

---

### Slide 6: 5. Server Engine & Multi-Tier Architecture

- **Visual:** Left: Server Component Tiers (I/O, Session, Lock Manager, Storage); Right: 5-step Server Processing Pipeline diagram.
- **Duration:** ~35 seconds
- **Speaker:** Presenter 1

> **[Spoken Script]**  
> *"The server is engineered across four decoupled internal tiers:  
> 1. The **I/O Multiplexer Layer**, which manages persistent TCP client sockets and dispatches incoming frames to a thread pool without blocking the network loop.  
> 2. The **Session & RBAC Interceptor**, maintaining an active connection registry mapping each socket to its authenticated user identity and group context.  
> 3. The **Concurrency & Lock Manager**, utilizing path-scoped `ReentrantReadWriteLock`s to allow multiple concurrent readers while enforcing exclusive access for writers.  
> 4. The **Dual Storage Layer**, combining an SQLite database for structured metadata and accounts with a structured directory tree under `/storage/groups/{groupId}/`.  
>  
> The 5-stage pipeline on the right illustrates the lifecycle of every incoming packet: frame assembly, security checking, lock acquisition, worker execution, and outgoing event dispatch."*

> **[Transition]**  
> *"I will now hand over to Long to present our core transport protocol and framing design."*

---

### Slide 7: 6. Protocol Design: Transport Choice & TCP Framing

- **Visual:** Left: Transport reasoning & Stream boundary dilemma; Right: Length-Type-Value (LTV) frame header layout with brace dimensions.
- **Duration:** ~40 seconds
- **Speaker:** Presenter 2

> **[Spoken Script]**  
> *"Thank you, Dong. Moving on to Section 6: Protocol Design, which is the heart of this report.  
>  
> For file sharing, reliability is non-negotiable. We selected **TCP** because it guarantees 100% byte delivery without packet loss, preserves strict in-order arrival, and provides native flow control.  
>  
> However, TCP operates as an unstructured byte stream—it does not preserve application message boundaries. In real network environments, we face two classic stream boundary challenges:  
> - **Fragmentation**, where a single message is split across multiple `read()` calls.  
> - **Concatenation**, where multiple consecutive messages are packed into a single network buffer.  
>  
> To solve this cleanly, we designed a **Length-Type-Value (LTV)** framing protocol. Every frame transmitted across the wire begins with a fixed **5-byte header**:  
> - A **4-byte Length field $N$**, encoded as a big-endian 32-bit unsigned integer, representing $1 + \text{payload bytes}$.  
> - A **1-byte Type field**, identifying the payload encoding.  
> - Followed by exactly $N - 1$ bytes of payload data.  
>  
> The receiver accumulates incoming bytes into an internal buffer and only triggers message parsing once all $4 + N$ bytes have arrived, completely eliminating stream boundary ambiguity."*

> **[Transition]**  
> *"Now let's examine the structure of our two frame types: Type 0x00 and Type 0x01."*

---

### Slide 8: 7. Framing Specification: Type 0x00 vs Type 0x01

- **Visual:** Top: Byte layout of Type 0x00 (JSON Control Frame); Bottom: Byte layout of Type 0x01 (Raw Binary File Chunk Frame).
- **Duration:** ~40 seconds
- **Speaker:** Presenter 2

> **[Spoken Script]**  
> *"A key innovation of our protocol is multiplexing both structured control commands and high-speed binary data over a **single persistent TCP connection** using two frame types:  
>  
> - **Type 0x00: JSON Control & Event Frame**.  
>   This frame carries UTF-8 encoded JSON strings. It is used for all metadata operations, including user authentication, directory queries, transfer handshakes, and asynchronous push events. It provides human-readable flexibility and schema evolvability.  
>  
> - **Type 0x01: Raw Binary File Chunk Frame**.  
>   Rather than encoding binary files into Base64—which introduces a 33% bandwidth overhead and heavy CPU serialization—we stream raw file chunks directly over the socket.  
>   As shown in the lower layout, following the 5-byte header, each Type 0x01 frame contains:  
>   - A **16-byte Transfer UUID**, matching the chunk to its staged upload session.  
>   - An **8-byte chunk Offset**, ensuring sequential file assembly and enabling resumable transfers.  
>   - A **4-byte ChunkLength**, specifying the size of the raw byte slice (typically 32 KiB or 64 KiB).  
>   - And finally, the raw file bytes themselves."*

> **[Transition]**  
> *"Let us see how Type 0x00 JSON messages are structured at the application layer."*

---

### Slide 9: 8. Message Envelope Schema

- **Visual:** Code snippets for Client Request, Server Response (Success & Error), and Server Push Event.
- **Duration:** ~35 seconds
- **Speaker:** Presenter 2

> **[Spoken Script]**  
> *"Every Type 0x00 message conforms to a unified envelope schema:  
> - A **Client Request** includes the protocol version `v`, message `kind: "REQUEST"`, a client-generated UUID `reqId`, the operation `cmd`, and a parameter `payload`.  
> - A **Server Response** echoes the same `reqId` and `cmd`. If the operation succeeds, `ok` is `true` and the resulting data is returned in `payload`. If the operation fails, `ok` is `false`, and a structured `error` object provides both a machine-readable error `code` and an informative `message`.  
> - Notice the lower-left envelope: **Server Push Events**. These are unsolicited notifications sent from server to client with `kind: "EVENT"`, such as notifying group members in real time when a new file has been added.  
>  
> Because each request is tagged with a unique `reqId`, clients can handle responses asynchronously without blocking other operations."*

> **[Transition]**  
> *"Next, let's explore our comprehensive application command catalog, starting with accounts and groups."*

---

### Slide 10: 9. Command Catalog: Authentication & Groups

- **Visual:** Table listing commands, caller roles, parameters, and responses for Auth & Group Management.
- **Duration:** ~35 seconds
- **Speaker:** Presenter 2

> **[Spoken Script]**  
> *"This table summarizes our authentication and group management commands:  
> - For authentication: `AUTH_REGISTER` creates accounts with salted password hashes; `AUTH_LOGIN` validates credentials and binds the connection session; `AUTH_LOGOUT` safely invalidates the session and cleans up resources; while `HEARTBEAT_PING` serves as a periodic keep-alive.  
> - For group management: Any authenticated user can issue `GROUP_CREATE` to start a new group and immediately become its Group Admin.  
> - Users discover groups via `GROUP_LIST` and apply to join using `GROUP_JOIN_REQ`.  
> - Crucially, only the Group Admin can call `GROUP_JOIN_REVIEW` to accept or reject applicants. Accepted users are promoted to the `MEMBER` role, triggering an automated event to the applicant."*

> **[Transition]**  
> *"Next, let us look at directory and file operations."*

---

### Slide 11: 10. Command Catalog: Directory & File Operations

- **Visual:** Table covering Directory Structure and File Metadata commands.
- **Duration:** ~35 seconds
- **Speaker:** Presenter 2

> **[Spoken Script]**  
> *"Section 10 details our directory and file metadata catalog:  
> - Members navigate workspaces using `DIR_LIST`, which returns child folders, file sizes, authors, and timestamps.  
> - Both Members and Admins can create subdirectories via `DIR_CREATE`.  
> - However, looking at `DIR_RENAME`, `DIR_DELETE`, `FILE_RENAME`, and `FILE_DELETE`: these operations explicitly require the **Admin Only** role.  
> If an unauthorized member sends these commands, the server rejects them with `FORBIDDEN_NOT_ADMIN`.  
> Furthermore, `DIR_DELETE` supports safe recursive directory cleanup, checked against our path lock manager to ensure no files are currently being transferred inside the target folder."*

> **[Transition]**  
> *"Now, let's examine the chunked file transfer lifecycle."*

---

### Slide 12: 11. Command Catalog: File Transfer Lifecycle

- **Visual:** Table covering Staged Upload, Binary Streaming, and Download commands.
- **Duration:** ~35 seconds
- **Speaker:** Presenter 2

> **[Spoken Script]**  
> *"Slide 11 shows how uploads and downloads are managed across both frame types:  
> - Uploading begins with a Type 0x00 `FILE_UPLOAD_INIT` request specifying the file path, total size, and target SHA-256 hash. The server validates quota and creates a hidden temporary staging file.  
> - The client then streams successive Type 0x01 binary chunks across the socket.  
> - Once all bytes are sent, the client issues `FILE_UPLOAD_FINISH`. The server verifies the complete SHA-256 hash and atomically moves the file into the active group repository.  
> - If an upload is aborted or interrupted, `FILE_UPLOAD_ABORT` cleans up the temporary file immediately.  
> - Downloads operate symmetrically: `FILE_DOWNLOAD_INIT` acquires a shared ReadLock on the file, the server streams Type 0x01 chunks back to the client, and `FILE_DOWNLOAD_ACK` releases the lock."*

> **[Transition]**  
> *"I will now hand over to Minh to present our state machines, concurrency control, and interaction sequences."*

---

### Slide 13: 12. State Machine: Client Connection Lifecycle

- **Visual:** 2-row TikZ state diagram showing client connection states and event transitions.
- **Duration:** ~40 seconds
- **Speaker:** Presenter 3

> **[Spoken Script]**  
> *"Thank you, Long. Now let us examine the state machines that guarantee system robustness.  
>  
> Slide 12 illustrates the **Client Connection Lifecycle**:  
> - The connection starts at `DISCONNECTED`.  
> - Upon establishing a TCP socket handshake, it enters `CONNECTED_UNAUTHENTICATED`. If no credentials arrive before a timeout, the connection is closed.  
> - A successful `AUTH_LOGIN` transitions the connection to `AUTHENTICATED`, binding the socket to an active user session.  
> - When the user selects a group workspace, the state moves to `GROUP_WORKSPACE`.  
> - From here, initiating an upload or download places the session in `TRANSFER_ACTIVE` while binary chunks stream. Once finished, it smoothly transitions back to `GROUP_WORKSPACE`.  
>  
> Most importantly, notice the **Socket Drop Guard**: if the physical TCP connection drops at any point, the server immediately triggers cleanup—unlinking temporary upload files, releasing all held read/write locks, and terminating the session registry entry."*

> **[Transition]**  
> *"Now let us examine our solution to the partial transfer challenge: the Resilient File Transfer State Machine."*

---

### Slide 14: 13. State Machine: Resilient File Transfer

- **Visual:** 2-row workflow diagram showing Staging, Receiving, Verification, Commit, and Rollback & Abort branches.
- **Duration:** ~45 seconds
- **Speaker:** Presenter 3

> **[Spoken Script]**  
> *"In Progress Report 01, we identified **Partial Transfers** as a critical challenge—broken network connections often leave corrupt, partial files on servers.  
>  
> Slide 13 presents our architectural solution:  
> - When `UPLOAD_INIT` is received, the server does **not** write to the destination directory. Instead, it enters `UPLOAD STAGING`, allocating a temporary staging file under `/storage/.tmp/.upload_{UUID}.tmp`.  
> - As Type 0x01 frames arrive in `RECEIVING CHUNKS`, the server verifies sequential offsets and computes a rolling SHA-256 digest on the fly.  
> - Once all bytes arrive, it enters `VERIFY INTEGRITY`, comparing the computed digest against the client's declared hash.  
> - Only if the checksum matches 100% does the server transition to `COMMITTED`, performing an **atomic filesystem rename** from `.tmp` into the active group folder and registering the metadata in SQLite.  
> - If an error occurs—such as disk full, socket disconnection, or a checksum mismatch—the system branches immediately to `ROLLBACK & ABORT`. The temporary staging file is deleted immediately.  
>  
> This guarantees a zero-corruption policy: a file is either 100% complete and verified, or it does not exist on disk at all."*

> **[Transition]**  
> *"Next, let's explore how our message processing pipeline solves concurrent access conflicts."*

---

### Slide 15: 14. Message Processing & Concurrency Pipeline

- **Visual:** Left: 5-step Server Pipeline; Right: Hierarchical ReadWriteLock conflict arbitration diagram.
- **Duration:** ~40 seconds
- **Speaker:** Presenter 3

> **[Spoken Script]**  
> *"The second challenge from Report 01 was **Concurrent Access Conflicts**—specifically, what happens if an Admin renames or deletes a folder while other members are actively uploading or downloading files inside it?  
>  
> We solved this through **Hierarchical Path Locking** using Java's `ReentrantReadWriteLock`:  
> - When Member 1 and Member 2 download `/reports/A.pdf`, both acquire a **Shared ReadLock**. ReadLocks are shared, so concurrent downloads proceed in parallel at full speed.  
> - Now, suppose the Group Admin attempts to delete or rename `/reports/`. The admin request requires an **Exclusive WriteLock**.  
> - Because active ReadLocks are currently held by the downloading members, the Lock Manager blocks or safely rejects the admin request with a `409 LOCK_CONFLICT` error.  
>  
> The active download is protected, and the file system is never left in an inconsistent or crashed state."*

> **[Transition]**  
> *"To see these mechanisms in action, let us examine three interaction sequence diagrams."*

---

### Slide 16: 15. Sequence: Group Join Request & Admin Review

- **Visual:** Sequence diagram showing Applicant (Bob), Server Engine, SQLite DB, and Admin (Alice).
- **Duration:** ~35 seconds
- **Speaker:** Presenter 3

> **[Spoken Script]**  
> *"Our first sequence demonstrates our RBAC join workflow:  
> 1. Applicant Bob sends a `GROUP_JOIN_REQ` for Group 4.  
> 2. The server records a `PENDING` request in SQLite and immediately confirms receipt to Bob.  
> 3. Simultaneously, the server pushes an asynchronous `EVENT: JOIN_REQUESTED` to Admin Alice's client.  
> 4. Alice reviews the applicant and sends `GROUP_JOIN_REVIEW` with action `ACCEPT`.  
> 5. The server updates Bob's role to `MEMBER` in the database, confirms to Alice, and pushes a real-time `JOIN_ACCEPTED` notification to Bob.  
>  
> Bob can now access Group 4's workspace without reconnecting."*

> **[Transition]**  
> *"Next, let's trace a resilient chunked upload sequence."*

---

### Slide 17: 16. Sequence: Chunked Upload & Atomic Commit

- **Visual:** Sequence diagram showing Client, Server Demuxer, Staging Engine, and Group Repository.
- **Duration:** ~40 seconds
- **Speaker:** Presenter 3

> **[Spoken Script]**  
> *"Slide 16 details the full upload pipeline:  
> 1. The client sends a Type 0 `UPLOAD_INIT` frame with file size and hash.  
> 2. The server prepares a temporary file `.upload_UUID.tmp` and replies with `ok`, granting an upload session token and chunk size.  
> 3. The client streams Type 1 binary frames: Chunk 0 at offset 0, Chunk 1, and so on. The server appends bytes and updates its rolling SHA-256 digest.  
> 4. When streaming finishes, the client sends a Type 0 `UPLOAD_FINISH` frame.  
> 5. The server verifies the final digest against the declared hash, atomically renames the temporary file into the group repository, and acknowledges success.  
>  
> Notice that binary chunk streaming happens with zero base64 overhead, and disk commitment is strictly atomic."*

> **[Transition]**  
> *"Next, let's examine our conflict resolution sequence."*

---

### Slide 18: 17. Sequence: Concurrency Conflict Resolution

- **Visual:** Sequence diagram showing Member (Reader), Lock Manager, Server Storage, and Admin (Writer).
- **Duration:** ~35 seconds
- **Speaker:** Presenter 3

> **[Spoken Script]**  
> *"Slide 17 demonstrates how our concurrency manager arbitrates collisions in real time:  
> 1. A member issues `DOWNLOAD_INIT` for `/data/A.pdf`.  
> 2. The Lock Manager grants a shared `ReadLock`, and the server begins streaming chunks to the reader.  
> 3. Mid-stream, the Group Admin issues `DIR_DELETE(/data/)`.  
> 4. Server Storage requests an exclusive `WriteLock` on `/data/`.  
> 5. The Lock Manager detects that an active ReadLock is held on `/data/A.pdf`. It immediately denies the WriteLock and responds with `Error: 409 LOCK_CONFLICT (Resource busy)`.  
> 6. The member completes the download unimpeded, releases the ReadLock, and file integrity is completely preserved."*

> **[Transition]**  
> *"Finally, let us review our implementation roadmap and conclusion."*

---

### Slide 19: 18. Implementation Roadmap & Next Milestones

- **Visual:** Phased schedule: Left: Phases 1 to 3; Right: Phases 4 & 5.
- **Duration:** ~30 seconds
- **Speaker:** Presenter 3

> **[Spoken Script]**  
> *"Here is our phased project roadmap:  
> - **Phase 1 (Completed)**: System modeling, LTV framing specification, RBAC matrix, and command catalog design.  
> - **Phase 2 (Current Sprint, Weeks 3–4)**: Multithreaded TCP socket engine, LTV byte accumulator, SQLite schema, and user authentication.  
> - **Phase 3 (Weeks 5–6)**: Group management, join review workflows, subfolder CRUD, and admin privilege enforcement.  
> - **Phase 4 (Weeks 7–8)**: Type 0x01 binary chunk streaming, temporary staging file engine, rolling SHA-256 validation, and network drop recovery.  
> - **Phase 5 (Weeks 9–10)**: JavaFX desktop client GUI with transfer progress bars, concurrent multi-client stress testing, and final defense."*

> **[Transition]**  
> *"Let us conclude our presentation."*

---

### Slide 20: 19. Summary & Conclusion

- **Visual:** Left: Core Contributions in Report 02; Right: Addressed Challenges from Report 01.
- **Duration:** ~35 seconds
- **Speaker:** Presenter 3

> **[Spoken Script]**  
> *"To summarize, in Progress Report 02 we have established:  
> 1. A formal **LTV framing protocol** that cleanly solves TCP stream fragmentation and concatenation over a single persistent socket.  
> 2. A rigid **Role-Based Access Control model** guaranteeing team workspace isolation and exclusive admin protection against unauthorized deletion or renaming.  
> 3. A fault-tolerant **staging architecture** guaranteeing zero corrupted partial files.  
>  
> Most importantly, we directly resolved the three core challenges defined in Report 01:  
> - Concurrent access is managed through path-scoped `ReadWriteLock`s.  
> - Partial transfers are resolved through atomic staging and SHA-256 verification.  
> - Session state is maintained seamlessly across requests via connection-bound session tables."*

> **[Transition]**  
> *"This brings us to the end of our design review."*

---

### Slide 21: 20. Thank You & Q&A

- **Visual:** Official HUST theme closing slide (`\hustthankyou`).
- **Duration:** ~20 seconds
- **Speaker:** All Presenters / Presenter 1

> **[Spoken Script]**  
> *"Thank you very much, Professor and classmates, for your time and attention.  
> We welcome any questions, comments, or feedback on our system architecture and protocol design."*

---

## Tips for Delivering the Presentation

1. **Pacing**: Speak at a steady, conversational pace (~130 words per minute). Do not rush through the TikZ diagrams.
2. **Pointing / Cues**:
   - On **Slide 4 (RBAC Matrix)**, gesture towards the red crosses under Member for `DELETE` and `RENAME`.
   - On **Slide 7 & 8 (Framing)**, emphasize the **5-byte header** and explain how Type 0x01 avoids Base64 overhead.
   - On **Slide 13 & 14 (State Machines)**, point out the **Rollback branch** and explain how `.tmp` files are deleted on connection drops.
   - On **Slide 18 (Concurrency Conflict)**, emphasize how this directly fulfills the challenge from Report 01.
3. **Q&A Readiness**:
   - *Why single TCP connection instead of dual (control + data)?* Answer: Single connection with LTV framing eliminates socket synchronization complexity, firewall/NAT issues with secondary ports, and connection overhead.
   - *Why not WebSocket or HTTP/2?* Answer: IT4062E requires building standard TCP socket programming from scratch (`java.net.Socket`), demonstrating custom protocol framing and I/O multiplexing.
