# State Convergence & Multi-Region Topologies

Global real-time apps require ultra-low latency, meaning users in Asia must connect to servers in Asia, and users in Europe to servers in Europe. Maintaining consistent state across these active-active regions introduces immense complexity.

## Conflict-Free Replicated Data Types (CRDTs)
Traditional distributed systems rely on Consensus Protocols or distributed locking (e.g., Raft, Paxos) to serialize changes. This requires high-latency round-trips to master centers.

When architecting real-time systems acting on shared state (e.g., collaborative text editing, global shopping carts, live telemetry):
1. **Model the Data Mathematically:** Represent the data using CRDT structures (like G-Counters, OR-Sets or LWW-Registers). 
2. **No Central Locks:** Because these structures are rigorously designed to be commutative and associative, regional databases can accept local write operations instantly with zero network locking.
3. **Eventual Deterministic Convergence:** When regional data centers exchange the operations asynchronously in the background, the mathematical guarantees of the CRDT force every node to merge and arrive at the exact same deterministic state.

## Edge Termination and Anycast
The public internet is highly volatile. Long-lived streaming TCP/QUIC connections risk higher packet loss the further they travel over open ISP links.

**The Solution:**
1. **Anycast Routing:** Bind your global fleet to Anycast IPs. The client resolves a single IP but the border routers direct the physical packet to the closest edge data center.
2. **TLS Termination:** Terminate the connection and perform the initial handshake at this literal edge node. 
3. **Internal Backhaul:** Tunnel the payload from the Edge Node over a dedicated, highly reliable, and optimized cloud backbone (the Relay Service Fan-Out network you designed) down to the appropriate physical pod. 
