# The MCP Gap

The architectural chasm between stateless tool invocation and stateful multi-agent execution: while the Model Context Protocol (MCP) standardizes how a single model calls external tools and reads raw data via JSON-RPC, and A2A/ACP standardize how agents pass conversational messages to one another, no layer in the current stack governs how structured working state, dependency graphs, and verified mutations persist and travel across agent boundaries.

Because that middle layer is missing, every multi-agent handoff today is forced to treat state as unstructured prose—turning distributed cognition into a high-token game of telephone.

## Where the Current Protocol Stack Breaks Down

| Protocol Layer | Primary Scope | Wire Primitive | What It Solves | What It Leaves Broken (The Gap) |
|---|---|---|---|---|
| MCP (Model Context Protocol) | Vertical I/O (Model ↔ Tools & Data) | Stateless JSON-RPC 2.0 (tools/call, resources/read) | Gives a single model hands and eyes to query APIs, read files, and run scripts. | Zero state continuity. Tool side effects happen blindly outside the protocol; when a session drops or hands off, the mental model built from those reads vanishes. |
| A2A (Agent-to-Agent) | Horizontal Tasking (Agent ↔ Agent) | Task envelopes, natural-language prompts, status polling | Lets an orchestrator delegate a prompt to a sub-agent and await a text/artifact reply. | The Cold-Start Tax. The receiving agent gets a text prompt, not the upstream working state; it must re-run MCP reads from scratch to rebuild context. |
| ACP (Agent Communication Protocol) | Discovery & Ontology (Platform ↔ Platform) | Capability registries, routing metadata, semantic schemas | Helps heterogeneous agents find each other and agree on message schemas. | No transactional guarantees. Agreeing on message vocabulary does not prevent two agents from clobbering the same worktree or hallucinating state drift. |
| The Missing Layer (State & Execution Fabric) | Horizontal State Sync (Worktree ↔ Worktree) | Content-addressed DAGs, scoped state deltas, transactional receipts | Moves verified working state—not prose—across agents and context windows. | This is the exact gap ARFP (arfp-exact-delta-v1) is architected to close. |

## The Four Structural Failures Inside the "MCP Gap"

When you try to build a serious multi-agent system (or a federated control plane) using only MCP and A2A, you hit four wall-to-wall systems bottlenecks:

### 1. The Cold-Start Rebuild Tax (Stateless RPC vs. Portable State)

MCP is fundamentally a client-server Remote Procedure Call pipe. Agent A calls resources/read and tools/call twenty times, ingesting raw files into its ephemeral context window and synthesizing an internal dependency graph inside its volatile KV cache.

 * The Failure: The moment Agent A hits a context-window ceiling ("window too small"), crashes, or delegates a sub-task to Agent B, all of that structured synthesis dies with the session.
 * The Consequence: Agent B receives a prose summary over A2A and has to fire its own MCP tool calls to re-read the codebase, re-discover the architecture, and rebuild the working state from zero. You pay quadratic token and latency costs for linear work.

### 2. Unbounded Side Effects Without ACID or Compare-and-Swap (CAS)

When an agent modifies the world through an MCP tool—writing to disk, mutating a database, or patching code—that mutation happens as an untracked, immediate side effect.

 * The Failure: MCP has no native concept of a pinned baseline hash, a staged transaction, a scoped mutation contract, or atomic Compare-and-Swap.
 * The Consequence: If you dispatch three parallel sub-agents connected to the same MCP filesystem or repo server, they overwrite each other's changes in real time. There is no protocol-level gate to verify that Agent B's output still applies cleanly to the baseline state Agent A started from.

### 3. The Context Serialization Dilemma (Full Dump vs. Lossy Prose)

Without a state-transport protocol sitting above MCP, an orchestrator handing work to a specialist has only two ways to pass context:

 * Stuff the entire upstream transcript and raw MCP tool outputs into the sub-agent's prompt, which immediately blows out token budgets, degrades attention, and leaks sensitive files across agent boundaries.
 * Ask the upstream model to write a natural-language summary of what it learned, which introduces silent omissions, lossy compression drift, and hallucinated assumptions.
 * What's Missing: The stack lacks Scoped Dependency DAG Slicing—the ability to ship only the exact, hash-verified slice of the working tree and interface contracts that a sub-agent needs to execute its bounded task, without shipping the whole repo or relying on prose summaries.

### 4. The Provenance and Learning Void (Chat Logs vs. State Trajectories)

When an MCP + A2A workflow finishes, the artifact left behind is an unstructured transcript of tool calls and chat messages.

 * The Failure: You cannot deterministically audit which agent introduced a subtle regression, you cannot cryptographically prove export-control or clean-room boundaries, and you cannot automatically extract clean training signal.
 * The Consequence: To get a system that "trains off itself" or enforces zero-trust federation across competing labs, you need every handoff to emit a structured, verifiable state transition—binding the exact baseline state, the bounded contract, the resulting worktree delta, and the deterministic validation outcome.

## How to Frame This Cleanly (The Two-Paragraph Synthesis)

If you want to articulate the exact boundary between MCP, A2A, and your state fabric without exposing internal schema mechanics, here is the clean framing:

> MCP gives an agent hands to touch tools; A2A gives agents a voice to pass messages; neither gives agents a shared, verifiable memory of the work in progress. When an agent reads a codebase or environment through MCP, the resulting structural understanding lives entirely inside that model's ephemeral context window. If that session drops or hands off a subtask to another model over A2A, the understanding evaporates—forcing the receiving agent to either trust a lossy prose summary or re-query the world through MCP from scratch, while any tool mutations happen as untracked side effects with zero conflict protection.

> The missing layer is a deterministic state-transport fabric that sits alongside MCP and A2A. Instead of shipping conversational transcripts or letting sub-agents blindly mutate shared tools, the fabric packages the working state as a scoped, content-addressed dependency graph and exchanges exact worktree deltas wrapped in transactional contracts. Agents only reconstruct the precise subgraph they need, every returned mutation is validated against a pinned baseline before advancing the canonical state, and every accepted transition leaves behind a verifiable audit and training trail.

## Interactive Architecture Comparison: The MCP/A2A Chasm vs. The State Fabric Bridge

Use the toggle and step controls below to compare what happens during a multi-agent handoff under today's Stateless MCP + A2A Stack versus closing the gap with a Deterministic State Fabric:

> Why This Framing Lands: When you describe the gap this way, nobody can argue that "MCP or A2A already does this." MCP is I/O (syscalls to tools), A2A is IPC (message passing between processes), and what you are building is the transactional filesystem and virtual memory manager that lets multiple cognitive processes share state without corrupting each other.
