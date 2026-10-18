# ARFP — Adaptive Reconstruction Fabric Protocol

**ARFP** (Adaptive Reconstruction Fabric Protocol, profile `arfp-exact-delta-v1`) is Dino Silvestri's state-transfer layer for multi-agent systems. It is deterministic, not neural: working state is externalized as canonical JSON (RFC 8785), every object is SHA-256 content-addressed, and what crosses the wire is an exact worktree delta against a pinned baseline hash plus a dependency DAG. The receiver reconstructs bit-exact, hash-verified state.

The open agent protocols — MCP, A2A, ACP — move messages. None of them moves working state. ARFP is the missing layer that sits beside them: **Target** (pinned baseline hash) → **Contract** (permitted mutation scope and invariants) → **Plan** (dependency DAG) → **Worktree Payload** (untrusted delta, carried in OFFER / RESULT envelopes) → **Validation Approval** (hash, schema, conflict and boundary checks) → **Commit Receipt** (signed sign-off that advances the head). State never advances by default.

## Documents

- **[ACP-in-ARFP, Formalized](ACP-in-ARFP-Formalization.md)** — The binding taken down to implementable artifacts: the dual-plane payload, a closed six-type schema for semantic memory nodes (GroundedFact, WorkingHypothesis, NegativeConstraint, Constraint, ToolLease, DeadEnd) with the evidence-digest rule that keeps inferred memory from masquerading as grounded truth, the MCP staging wrapper that captures reads as evidence and stages writes until a Commit Receipt releases them, a worked handoff, compare-and-swap concurrency and the receipt chain, and the open problems stated plainly.
- **[The MCP Gap](The-MCP-Gap.md)** — The architectural chasm between stateless tool invocation and stateful multi-agent execution: where MCP, A2A and ACP each break down, the four structural failures (cold-start rebuild tax, unbounded side effects without ACID/CAS, the context serialization dilemma, the provenance and learning void), and the clean two-paragraph framing of the missing state layer.
- **[Binding ACP to ARFP: Flow-State Memory](Binding-ACP-to-ARFP.md)** — Why ACP and ARFP need each other: ACP governs meaning without state, ARFP governs state without cross-platform ontology. The unified four-protocol pipeline (MCP ingestion → ACP semantic typing → ARFP crystallization → A2A dispatch), three mechanics the binding unlocks (negative-space persistence, stateful MCP pipeline forwarding, ACP-driven adaptive DAG slicing), the dual-plane state object, and two spec edge cases (ontology hash pinning; GroundedFact vs. WorkingHypothesis).
- **[Voice-Native Agents: Feedback for Anthropic and the ARFP Layer](Voice-Native-Agents-Feedback-for-Anthropic-and-the-ARFP-Layer.md)** — The feedback document that motivates ARFP from the product side: eight feature asks for voice-native agents (lead with resolving vague references to concrete code), the protocol-stack analysis, the ARFP design at a glance, federation under one control plane, and honest pushback with open problems.

## Status

Public repo in progress (October 2026). Architecture pieces and handoff docs exist; internals are deliberately not detailed here.
