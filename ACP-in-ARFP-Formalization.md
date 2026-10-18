# ACP-in-ARFP, Formalized: The Bound State Object and the MCP Staging Wrapper

This document takes the ACP × ARFP binding from [Binding ACP to ARFP](Binding-ACP-to-ARFP.md) down to the two artifacts the binding needs before anyone can implement it: **(1)** a concrete JSON schema for the semantic memory nodes that live on ARFP's Semantic Plane, and **(2)** the MCP staging wrapper that captures tool reads and writes into the Dependency DAG instead of letting them happen as untracked side effects. It closes with a worked handoff, the concurrency rule, and the failure modes that remain open.

The one-sentence version of the whole stack, unchanged: **MCP ingests the world, ACP types the meaning, ARFP locks the meaning and the worktree deltas into a lossless Merkle DAG, and A2A routes the execution.**

## 1. The Dual-Plane Payload, Concretely

An ARFP Worktree Payload carries two planes, both canonicalized with RFC 8785 (JSON Canonicalization Scheme) and hashed with SHA-256 before they enter the DAG:

- **Syntactic Plane** ($\Delta_{Artifact}$) — exact file/code/data mutations, governed by `arfp-exact-delta-v1` as already specified.
- **Semantic Plane** ($\mathcal{S}_{ACP}$) — the flow state: a set of typed memory nodes, each described by an ACP ontology schema, each carrying its own digest.

The planes are bound, not just bundled: semantic nodes reference syntactic objects by digest (a `GroundedFact` about a function points at the exact file version it was observed in), and the Payload's root hash is computed over both plane roots. A receiver cannot accept one plane and silently drop the other; validation covers the pair.

```jsonc
// Worktree Payload (excerpt) — dual-plane root
{
  "payload_id": "sha256:9f2c…",
  "target_hash": "sha256:41ab…",            // pinned baseline (Target)
  "syntactic_root": "sha256:77d0…",          // Merkle root of file deltas
  "semantic_root": "sha256:b31e…",           // Merkle root of ACP memory nodes
  "acp_ontology_sha256": "sha256:0c4d…",     // pinned ontology version — see §4
  "planes_bound": true
}
```

## 2. The Semantic Memory Node Schema

Every node on the Semantic Plane is one of a small set of ACP types. The set is deliberately closed — an open type system would reintroduce prose through the back door. Six node types cover the flow state described in the binding document:

| Node type | Carries | Trust rule |
|---|---|---|
| `GroundedFact` | An assertion + the SHA-256 digest of the raw MCP tool output that proved it | May be relied on by any receiver |
| `WorkingHypothesis` | An assertion + the reasoning path that produced it | Provisional until a Validation Approval promotes it |
| `NegativeConstraint` | A falsified approach + the evidence that falsified it | Inherited on token zero; the pruned search tree |
| `Constraint` | A rule the work must respect, flagged `hard` or `soft` | `hard` violations fail validation; `soft` violations must be reported in the RESULT |
| `ToolLease` | A live MCP resource state: snapshot digest, cursor, staged mutations | Receiver may continue the pipeline without re-querying |
| `DeadEnd` | A route explored and abandoned without a formal falsification (timeout, cost, interruption) | Weaker than `NegativeConstraint`; receiver may re-try if its profile differs |

The `GroundedFact` / `WorkingHypothesis` split is the load-bearing rule from the binding document, made mechanical: **a node may only claim `GroundedFact` if its `evidence_digest` resolves to an MCP output captured by the staging wrapper (§3).** Hashing proves a node wasn't altered on the wire; the evidence digest is what proves it wasn't hallucinated at the source. A validator rejects a `GroundedFact` whose evidence digest is missing or unresolvable — no exceptions, no "trust me" flag.

```jsonc
// Semantic memory node — GroundedFact example
{
  "node_id": "sha256:5e1a…",
  "type": "GroundedFact",
  "ontology": "acp.core/1.3",                 // human-readable version…
  "statement": "District join fails when the two district graphs share no node id at the seam.",
  "about": ["sha256:77d0…"],                   // syntactic digests this fact describes
  "evidence_digest": "sha256:aa09…",           // raw MCP read output, staged in §3
  "observed_at_state": "sha256:41ab…",         // baseline the observation was made against
  "confidence": "observed"                     // observed | derived | reported
}

// NegativeConstraint example — the dead end a receiver must not re-walk
{
  "node_id": "sha256:61bc…",
  "type": "NegativeConstraint",
  "statement": "Reindexing the road graph does not fix the seam; ids are stable across reindex.",
  "falsified_by": "sha256:5e1a…",              // the GroundedFact that killed it
  "cost_to_falsify_tokens": 18400              // optional; lets routers price re-exploration
}
```

Two schema disciplines keep the plane honest as it scales:

- **Content addressing all the way down.** Node ids are digests of the canonical node body. Two agents that independently derive the same fact produce the same node id, which makes deduplication a hash comparison instead of a judgment call — the merged layer gets richer without getting redundant.
- **No free text where a reference exists.** Statements are human-readable (they have to be — a person audits this), but anything the statement points *at* — a file, a fact, a tool output — is a digest, not a description. Prose is the annotation, never the pointer.

## 3. The MCP Staging Wrapper

The staging wrapper is what turns MCP from a side-effect pipe into the fabric's ingestion layer. It sits between the agent and every MCP server and enforces three rules:

1. **Every read is captured.** The raw output of `resources/read` and `tools/call` is canonicalized, hashed, and stored as an evidence object in the DAG. When the agent later asserts a `GroundedFact`, its `evidence_digest` points here. Reads are cheap to keep because identical outputs collapse to one content address.
2. **Every write is staged, never executed.** A `tools/call` that would mutate the world is recorded as a *staged mutation* against the pinned baseline — the dry-run delta, its preconditions, and the tool lease it belongs to. The real MCP server is not touched. Staged mutations ride inside the Worktree Payload's Syntactic Plane.
3. **Commit is a protocol event, not a tool call.** Staged mutations reach the real world only when the ARFP sequence completes: Validation Approval checks the RESULT against the Contract (scope, invariants, baseline still current via compare-and-swap), and the Commit Receipt authorizes the wrapper to replay the staged writes against the actual MCP servers — in DAG order, aborting the whole batch if any write's precondition has gone stale.

```jsonc
// ToolLease — a hot MCP pipeline handed across the agent boundary
{
  "node_id": "sha256:83f7…",
  "type": "ToolLease",
  "server": "mcp://godot-project",
  "resource": "res://content/base/roadgraph.json",
  "snapshot_digest": "sha256:41ab…",           // the exact bytes both agents agree they saw
  "cursor": { "kind": "ast-path", "value": "/root/districts" },
  "staged_mutations": ["sha256:d204…"],        // dry-run only until a Commit Receipt lands
  "lease_expires_at_state": "sha256:41ab…"     // lease dies if the baseline advances past it
}
```

The lease expiry rule matters: a `ToolLease` is only meaningful against the state it was captured in. If the canonical head advances, the lease's snapshot digest no longer matches the world, and the receiver must re-read through MCP rather than inherit a stale cursor. This is the one place the fabric deliberately pays the cold-start cost — a stale lease inherited silently is worse than no lease.

## 4. The Two Boundary Rules, Made Mechanical

From the binding document, restated as validator behavior:

- **Ontology pinning.** Every Contract carries `acp_ontology_sha256`, the digest of the exact ACP schema definitions used to type the Semantic Plane. A worker that does not hold that exact schema rejects the OFFER *before* mounting anything. Schema negotiation happens in the ACP capability exchange upstream; by the time ARFP is involved, the version question is already settled and hashed. Near-enough ontology matching is how reconstructions drift; the rule exists because "almost the same schema" and "a different schema" produce the same failure, just slower.
- **Grounded vs. inferred.** The validator enforces the type split from §2 at RESULT time as well as OFFER time: a returning agent cannot upgrade its own `WorkingHypothesis` to a `GroundedFact` without attaching an evidence digest captured by *its* staging wrapper during the task. Self-promotion of hypotheses is the quiet corruption path in every multi-agent system that shares notes; the schema closes it structurally instead of by policy.

## 5. A Worked Handoff

The scenario from the feedback document: a debugging session on a Godot city map, where the district join breaks between two ends of the city. Agent A (architect mode) has been driving; its context ceiling approaches; the task routes to Agent B (a different model family, stronger at code inspection).

**Today (MCP + A2A only):** A writes a prose summary. B re-reads the repo through MCP, re-derives the district graph structure, and — because the summary didn't mention it — spends its first twenty minutes re-testing the reindexing theory A already falsified. A's open tool reads die with its session.

**With the binding:** A's staging wrapper has captured every MCP read as evidence objects. A's flow state crystallizes into a Semantic Plane: two `GroundedFact`s about the seam (with evidence digests), one `NegativeConstraint` (reindexing, falsified, cost recorded), one `WorkingHypothesis` (the seam expects mirrored node ids), and a `ToolLease` on the road-graph resource. ARFP slices the DAG to B's ACP capability manifest — B gets the district subgraph, the seam contracts, and digest stubs for the rest of the city. A2A dispatches a task handle referencing the mounted state digest. B starts at token zero knowing what is proven, what is guessed, and what not to retry; its staged fix returns as a RESULT, validates against the Contract and the pinned baseline, and the Commit Receipt advances the head — leaving the (Contract, Plan, Delta, Validation) tuple behind as audit trail and training signal.

## 6. Concurrency and the Receipt Chain

Parallel specialists are the normal case, so the fabric's rule is compare-and-swap at the head: a RESULT validates only if the baseline it was computed against is still the canonical head, or if its delta re-bases cleanly under the Contract's conflict rules. Two agents may work the same baseline concurrently; they may not both advance it silently. The loser of a compare-and-swap race isn't discarded — its delta, validation record, and the winning delta form exactly the contrastive pair the control plane trains on. Failure is data, receipted like everything else.

The Commit Receipts chain: each receipt references the previous head, the delta applied, and the validation record. The chain is the audit log — replayable, attributable, and sufficient to prove which agent introduced which state transition, which is what makes clean-room and export-control boundaries provable after the fact instead of arguable.

## 7. What Remains Open

Carried forward honestly from the source documents, plus what this formalization exposed:

- **Pre-OFFER semantic classification.** Hashes are content-blind; nothing in the envelope stops a malicious or confused agent from typing a harmful instruction as a `Constraint`. Classification has to happen before an OFFER is composed, and the fabric doesn't yet specify by whom. (Open problem in the feedback document's pushback table; still open.)
- **Path-topology leakage.** Canonical paths inside deltas reveal repo structure to receivers who only needed a slice. Canonical path aliasing per-OFFER is the sketched fix; the aliasing scheme is not yet specified.
- **Lease granularity.** §3 treats a lease as one resource snapshot. Live pagination streams and multi-resource transactions need either composite leases or an explicit exclusion from forwarding. Undecided.
- **Soft-constraint reporting format.** `Constraint: soft` requires violations to be "reported in the RESULT," but the report shape isn't yet an ACP type. It should be — otherwise soft-constraint reporting degrades into prose, which is the disease the whole stack exists to cure.

## Next artifacts

Two documents would take this from architecture to buildable spec: the full JSON Schema files for the six node types and the six spine objects (machine-checkable, versioned with the ontology hash), and a staging-wrapper reference implementation against a real MCP server, measured against the cold-start baseline it replaces.
