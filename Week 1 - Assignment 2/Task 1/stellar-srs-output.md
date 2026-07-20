# Software Requirement Specifications (SRS)
## Stellar Consensus Protocol (SCP) Engine

---

## 1. Introduction

### 1.1 Purpose
This document specifies the software requirements for the **Stellar Consensus Protocol (SCP) Engine**, a core consensus component operating within a Federated Byzantine Agreement System (FBAS). The engine enables decentralized, open-membership consensus across independent network nodes without requiring a centralized membership registry or proof-of-work resource consumption.

This specification bridges high-level cryptographic and theoretical system guarantees with actionable implementation requirements for software engineers, systems architects, and core protocol maintainers (`stellar-core`).

### 1.2 Scope
The SCP Engine provides asynchronous, low-latency transaction and state agreement across distributed nodes. The engine is responsible for:

- Managing node-specific trust decisions via Quorum Slices.
- Executing a Federated Nomination Sub-Protocol to establish candidate values.
- Executing a Federated Ballot Sub-Protocol to commit or abort ballot values.
- Neutralizing stuck consensus states using monotonic ballot counters.
- Reaching safety and liveness guarantees across non-uniform, decentralized network topologies.

> *Out of Scope:* Top-tier network peer-to-peer transport protocol framing (gRPC/TCP overlay), physical ledger storage engines (e.g., SQL/RocksDB integration), and transaction signature verification prior to nomination ingestion.

---

## 2. Overall Description

### 2.1 Product Perspective
The SCP Engine sits directly between the Peer-to-Peer (P2P) Overlay Network Layer and the Replicated State Machine / Ledger Application Layer (`Herder` / `LedgerManager`). Unlike traditional centralized Byzantine Fault Tolerant (BFT) systems (e.g., PBFT) that require a static, unanimously agreed membership list, or Proof-of-Work (PoW) mechanisms that tie safety to hash power, SCP operates over an FBA model. Each participating node independently configures its trust parameters, which collectively synthesize global network quorums.

+-------------------------------------------------------+
|          Ledger Manager / State Machine               |
+-------------------------------------------------------+
^
| (Externalized Ledger Values)
+--------------------------v----------------------------+
|                       SCP Engine                      |
|  +--------------------+      +---------------------+  |
|  | Nomination Sub-Prot | ---> | Ballot Sub-Protocol |  |
|  +--------------------+      +---------------------+  |
|  +-------------------------------------------------+  |
|  |           Quorum & Slice Evaluator              |  |
|  +-------------------------------------------------+  |
+-------------------------------------------------------+
^
| (Signed XDR Protocol Messages)
+--------------------------v----------------------------+
|                   P2P Overlay Layer                   |
+-------------------------------------------------------+

### 2.2 Product Functions
The core functions performed by the SCP Engine include:

- **Quorum Slice Discovery & Tracking:** Dynamically evaluates network quorum availability and intersection based on local node trust definitions $Q(v)$.
- **Federated Voting Execution:** Manages three-phase voting state transitions (`vote`, `accept`, `confirm`) to ensure two intact nodes never externalize contradictory statements.
- **Candidate Nomination:** Aggregates and filters proposed slot values from peer nodes to converge on a single composite value.
- **Ballot Commitment & Neutralization:** Prepares and commits ballot pairs $\mathbf{b} = \langle n, x \rangle$ while incrementing ballot counters $n$ to recover from delayed or interrupted consensus rounds.

### 2.3 User Classes and Characteristics

| User / Node Class | Description & Operational Characteristics |
| :--- | :--- |
| **Tier-1 Validators** | Widely trusted, highly available core validator nodes operated by major network infrastructure providers. Form the backbone of network quorum intersection. |
| **Tier-2 Validators** | Regional or organizational validators that build quorum slices relying on Tier-1 nodes and direct peers to validate state transitions. |
| **Watcher Nodes** | Non-validating nodes that observe consensus and verify externalized transactions without providing quorum weight to upper tiers. |
| **Application Services** | Client Horizon API instances, RPC nodes, and smart contract execution contexts consuming externalized ledger states. |

---

## 3. System Features

### 3.1 Federated Nomination Sub-Protocol

#### 3.1.1 Description and Priority

- **Priority:** High
- **Description:** For every target slot $i$, the Nomination Sub-Protocol processes input values proposed by the node and its peers. It produces one or more candidate values and combines them into a single deterministic composite value $z$.

#### 3.1.2 Functional Requirements

- `FR-NOM-01`: The system **shall** maintain local nomination state per slot $i$, comprising:
  - $X$: The set of values the local node $v$ has voted to nominate.
  - $Y$: The set of values the local node $v$ has accepted as nominated.
  - $Z$: The set of values the local node $v$ considers confirmed candidate values.
  - $N$: The latest `NOMINATE` message payload received from each peer node.
- `FR-NOM-02`: The system **shall** compute a deterministic priority for peer nodes per round $n$ using the slot-specific hash function:
  $$\text{priority}(n, v') = G_i(P, n, v')$$
  where $G_i$ derives from SHA-256 over slot state dependencies.
- `FR-NOM-03`: The system **shall** vote to nominate values proposed by the neighbor node $v_0$ possessing the highest priority in $\text{neighbors}(v, n)$.
- `FR-NOM-04`: Upon confirming at least one candidate value ($Z \neq \emptyset$), the system **shall** immediately cease voting to nominate any new values for slot $i$.
- `FR-NOM-05`: The system **shall** continue accepting `nominate` statements from peers even after $Z \neq \emptyset$ if accepted by a $v$-blocking set.
- `FR-NOM-06`: The system **shall** derive the composite candidate value $z$ using a deterministic combination function:
  $$z = \begin{cases} \text{combine}(Z) & \text{if } Z \neq \emptyset \\ \text{combine}(Y) & \text{if } Z = \emptyset \text{ and } Y \neq \emptyset \\ \text{combine}(X) & \text{otherwise} \end{cases}$$

---

### 3.2 Federated Ballot Sub-Protocol

#### 3.2.1 Description and Priority

- **Priority:** High
- **Description:** The Ballot Sub-Protocol commits or aborts ballots of the form $\mathbf{b} = \langle n, x \rangle$, where $n \ge 1$ is a counter and $x$ is the composite slot value. It guarantees that intact nodes externalize exactly one value per slot without getting stuck in dead-end states.

#### 3.2.2 Functional Requirements

- `FR-BAL-01`: The system **shall** transition through three distinct operational phases per slot:
  1. `PREPARE`: Prepare candidate ballots and vote to abort lower incompatible ballots.
  2. `CONFIRM`: Confirm that a target ballot is prepared and vote to commit.
  3. `EXTERNALIZE`: Finalize state commitment and broadcast completion proof (`Stellar-SCP.x`) to the network.
- `FR-BAL-02`: While in the `PREPARE` phase, the system **shall** vote to abort all lower incompatible ballots $\mathbf{b}_{old} < \mathbf{b}$ where $\mathbf{b}_{old}:x \neq \mathbf{b}:x$ before voting to commit ballot $\mathbf{b}$.
- `FR-BAL-03`: The system **shall** track the two highest incompatible accepted prepared ballots $(p, p')$ to subsume past prepared statements.
- `FR-BAL-04`: A node **shall** set its commit range boundaries $[c, h]$ such that $c \le h$ represents the lowest and highest ballots for which the node has voted to commit without accepting an abort.
- `FR-BAL-05`: The system **shall** maintain the invariant $c \le h \le \mathbf{b}$ whenever $c \neq 0$.
- `FR-BAL-06`: The system **shall** transition to `CONFIRM` upon accepting `commit` for one or more ballots.
- `FR-BAL-07`: The system **shall** transition to `EXTERNALIZE` upon confirming `commit` for a ballot $c$, emit an `EXTERNALIZE` message payload, and write value $c:x$ to the ledger state.
- `FR-BAL-08`: If consensus progress stalls, the system **shall** trigger a ballot timeout, increment the ballot counter $n \leftarrow n + 1$, preserve value $x = h:x$ (or update $x = z$ if $h = 0$), and attempt to prepare ballot $\langle n + 1, x \rangle$.
- `FR-BAL-09`: If a $v$-blocking set of peer nodes asserts ballot counters higher than $\mathbf{b}:n$, the system **shall** immediately advance its local counter $\mathbf{b}:n$ to match the lowest counter asserting $v$-blocking quorum alignment, bypassing local timer delays.

---

### 3.3 Quorum & Slice Management Engine

#### 3.3.1 Description and Priority

- **Priority:** High
- **Description:** Parses and evaluates local trust configurations $Q(v)$, maintaining real-time awareness of system quorums and $v$-blocking sets.

#### 3.3.2 Functional Requirements

- `FR-QRM-01`: The system **shall** allow each node $v$ to independently define its quorum function $Q(v) \subseteq 2^V \setminus \{\emptyset\}$, where $v \in q$ for all $q \in Q(v)$.
- `FR-QRM-02`: A subset of nodes $U \subseteq V$ **shall** be evaluated as a valid quorum if and only if $U \neq \emptyset$ and $U$ contains at least one slice for every member $v \in U$.
- `FR-QRM-03`: A set of nodes $B \subseteq V$ **shall** be identified as $v$-blocking for node $v$ if $B$ overlaps every slice in $Q(v)$ (i.e., $\forall q \in Q(v), q \cap B \neq \emptyset$).
- `FR-QRM-04`: The engine **shall** validate that local slice configurations maintain quorum intersection despite potential Byzantine failures in dispensable sets ($DSets$).

---

## 4. Non-Functional Requirements

### 4.1 Performance

- `NFR-PRF-01`: **Latency Target:** Under standard mainnet network conditions, the consensus engine **shall** achieve externalized consensus within nominal ledger closing intervals (averaging approximately 5 seconds).
- `NFR-PRF-02`: **Resource Overhead:** The consensus engine **shall** maintain a low computational footprint suitable for standard server hardware without requiring dedicated mining, cryptographic hashing acceleration, or high-power GPU units.
- `NFR-PRF-03`: **Message Compactness:** Protocol status representation **shall** utilize standard XDR serialization formats (`Stellar-SCP.x`) to optimize payload transmission across the overlay network.

> *Throughput Note:* Network capacity per slot is governed by configurable ledger transaction set limits (`max_tx_set_size`) and smart contract resource limits, enabling controlled scaling across operations per ledger.

### 4.2 Security & Identity

- `NFR-SEC-01`: **Digital Signatures:** All protocol message payloads (`NOMINATE`, `PREPARE`, `CONFIRM`, `EXTERNALIZE`) **shall** be cryptographically signed using Ed25519 public keys (`NodeID`).
- `NFR-SEC-02`: **Asymptotic Security:** System safety **shall** rely on standard cryptographic primitives (Ed25519 signatures and SHA-256 hashes) designed to resist brute-force cryptographic attacks.
- `NFR-SEC-03`: **Byzantine Fault Tolerance:** The system **shall** guarantee safety across asynchronous network delays, preventing intact nodes from externalizing divergent values provided well-behaved nodes maintain quorum intersection.
- `NFR-SEC-04`: **Sybil Resistance:** Decentralized trust slice configuration **shall** prevent Sybil attacks without requiring centralized admission control or token-staking slash mechanisms.

> *Identity Management Note:* Node identities (`NodeID`) are tied directly to static Ed25519 keypairs. Node key updates are propagated out-of-band by publishing updated node metadata files (`stellar.toml`) and adjusting quorum configurations across peer nodes.