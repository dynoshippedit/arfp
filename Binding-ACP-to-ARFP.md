# Binding ACP to ARFP: Flow-State Memory

Binding ACP (Agent Communication Protocol) to ARFP (Adaptive Reconstruction Fabric Protocol) transforms ephemeral agent "flow state" into a lossless, content-addressed memory fabric—closing the chasm between MCP tool reads and A2A task handoffs.

In humans and LLMs alike, peak "flow state" is not raw data; it is a tightly bound graph of active context, verified facts, live tool handles, and already-eliminated dead ends. Today, when an agent ingests data via MCP and hands off a task via A2A, that flow state evaporates because KV caches are architecture-locked and prose summaries are lossy. By using ACP as the cross-platform semantic type system and ARFP as the deterministic Merkle-DAG transport, one agent can freeze its exact cognitive working state and another agent—on a completely different model family—can mount it instantly without rebuilding context from scratch.

## 1. Why ACP and ARFP Need Each Other to Create "Flow-State Memory"

Neither protocol can bridge the MCP-to-A2A chasm alone because one governs meaning without state and the other governs state without cross-platform ontology:

 * **ARFP Without ACP (Artifacts Without Semantics):** On its own, arfp-exact-delta-v1 deterministically ships file worktrees, dependency DAGs, and SHA-256 deltas. That guarantees the receiving agent gets the exact files, but it does not standardize how heterogeneous platforms represent cognitive state—such as which hypotheses were falsified, which constraints are soft vs. hard, or what an upstream MCP tool observation actually means.
 * **ACP Without ARFP (Vocabulary Without Persistence):** ACP lets agents across different platforms discover capabilities and agree on shared semantic schemas (what words, intents, and data structures mean). However, ACP is still a stateless messaging layer; it has no baseline hash pinning, no delta compression, no Compare-and-Swap (C19) concurrency control, and no cryptographic Commit Receipts.
 * **The ACP × ARFP Binding (Lossless Semantic State):** When you bind them, ACP defines the normative schema for cognitive primitives, and ARFP canonicalizes (RFC 8785) and hashes (SHA-256) those primitives directly into the Dependency DAG. Working memory stops being a volatile stream of chat tokens and becomes a versioned, hash-verified data structure.

## 2. The Unified 4-Protocol Pipeline: From MCP Ingestion to A2A Execution

Here is how the complete pipeline operates when ACP and ARFP are fused inside the bridge between MCP and A2A:

| Pipeline Stage | Protocol Layer | Mechanical Role in the Flow-State Bridge | What Travels on the Wire |
|---|---|---|---|
| 1. Environment Ingestion | MCP (Model ↔ Tools) | Queries external APIs, databases, and filesystems; stages raw tool side effects instead of executing blind writes. | JSON-RPC resources/read and tools/call outputs captured with provenance hashes. |
| 2. Semantic Typing & Negotiation | ACP (Ontology & Capabilities) | Translates raw MCP observations and internal reasoning into standardized, cross-platform semantic types (Fact, Constraint, DeadEnd, ToolLease) and negotiates the target worker's capability profile. | Strongly typed semantic objects and capability descriptors agreed upon across agent platforms. |
| 3. State Crystallization & Slicing | ARFP (Deterministic State Fabric) | Wraps the ACP-typed cognitive nodes and file worktree deltas into an RFC 8785 canonicalized, SHA-256 Merkle DAG (OFFER envelope) sliced to match the worker's ACP profile. | Target hash + Contract + Plan DAG + Worktree Payload (semantic state + file deltas). |
| 4. Task Dispatch & Instant Mount | A2A (Agent ↔ Agent Routing) | Dispatches the execution signal to the specialist agent, which mounts the ARFP subgraph and begins execution inside the upstream agent's flow state. | Lightweight task handle referencing the mounted ARFP state digest; returns RESULT envelope. |

## 3. Three Breakthrough Mechanics Unlocked by the Binding

When you fuse ACP's semantic contract with ARFP's deterministic state graph, three specific problems that plague multi-agent pipelines disappear:

### A. Negative-Space Persistence (Inheriting Pruned Search Trees)

When you are in deep flow on a complex problem, half of your working memory consists of negative knowledge: the three approaches you already proved won't work and the edge cases you already ruled out.

 * In a standard MCP-to-A2A handoff, the sub-agent only gets the current prompt and immediately wastes 20,000 tokens re-exploring the exact same dead ends the first agent just abandoned.
 * In the ACP + ARFP binding, falsified paths are emitted as ACP-typed NegativeConstraint nodes pinned inside the ARFP Plan DAG. The receiving agent mounts the pruned search space deterministically—it knows what not to try on token zero.

### B. Stateful MCP Pipeline Forwarding (Zero-Requery Handoffs)

Normally, if Agent A opens a database cursor, inspects an AST, or pulls a live API pagination stream via MCP, Agent B cannot inherit that live connection; Agent B has to re-authenticate and re-read via MCP from scratch.

 * By binding ACP resource descriptors inside ARFP Worktree Payloads, the OFFER envelope passes hash-pinned MCP state leases (exact snapshot digests, pagination cursors, and staged dry-run mutations).
 * The MCP pipeline stays hot across the agent boundary, while ARFP's Validation Approval gate ensures that staged MCP tool writes only commit to the real world when the Commit Receipt is signed.

### C. ACP-Driven Adaptive DAG Slicing

This makes the "Adaptive" in ARFP completely mechanical rather than heuristic:

 * How does the Control Plane know which slice of the working state DAG to pack into an OFFER?
 * Before ARFP slices the graph, ACP performs capability negotiation. The receiving agent declares its ACP manifest (e.g., context ceiling, supported schema versions, domain specialization, security clearance).
 * The ARFP engine uses that ACP manifest as a deterministic filter over the Dependency DAG—computing the exact minimal subgraph closure required for that specific agent profile and stubbing out the rest as opaque SHA-256 digests.

## 4. Formalizing the Bound State Object (ACP-in-ARFP)

To make this concrete in code and spec, an ARFP Worktree Payload no longer just carries raw file diffs; it carries a Dual-Plane State Graph where both planes undergo RFC 8785 canonicalization and SHA-256 hashing:

 * The Syntactic Plane ($\Delta_{\text{Artifact}}$): Exact file, code, or data mutations governed by arfp-exact-delta-v1.
 * The Semantic Plane ($\mathcal{S}_{\text{ACP}}$): The structured "flow state"—typed via ACP ontologies so Claude, Kimi, or a local model parses the exact same operational constraints without relying on natural-language interpretation.

## 5. Two Hard Edge Cases to Lock Down in the Spec

To keep the binding airtight against adversarial or noisy models, the specification has to enforce two boundary rules:

 * **Pinning Ontology Versions Against Hash Drift:** If two platforms use slightly different versions of an ACP semantic schema, their local reconstructions will diverge. Rule: Every ARFP Contract object must include an immutable acp_ontology_sha256 field. If a worker node does not possess the exact hashed ACP schema definition, it must reject the OFFER prior to execution rather than guessing field meanings.
 * **Separating Grounded MCP Truth from Inferred Memory:** Cryptographic hashing proves a memory node wasn't altered on the wire (C03), but it doesn't prove the upstream model didn't hallucinate an assertion during its flow state. Rule: The ACP schema must enforce a strict type split between GroundedFact (which must attach the SHA-256 digest of the raw MCP tool output that proved it) and WorkingHypothesis (which downstream agents treat as provisional until verified by an ARFP Validation Approval gate).

Rest easy—the core insight is locked in right here: MCP ingests the world, ACP types the meaning, ARFP locks the meaning and worktree deltas into a lossless Merkle DAG, and A2A routes the execution. Whenever you pick this back up, would you like to formalize the exact ACP-in-ARFP JSON schema for the semantic memory nodes, or map out how the MCP staging wrapper captures tool reads into the DAG?
