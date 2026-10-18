# Voice-Native Agents: Feedback for Anthropic and the ARFP Layer

Oct 18, 2026 · @Dino Silvestri

## Summary

Voice is the native interface for people who direct AI rather than type to it, and today Claude's voice surface cannot see a file, hear an audio file, reach the user's machine, or run an agentic loop. Competitors are shipping those from a voice session now. The second half of this document argues the deeper fix: the open agent protocols (MCP, A2A, ACP) move messages but not working state, and a state-transfer layer (ARFP) plus federated specialist models under one control plane is where the next advantage sits. Whoever owns the control plane owns the user.

## The gap, in one night

On October 18, 2026, one voice session with Claude Fable 5.1 surfaced four hard limits. None of them were reasoning failures; all of them were plumbing.

| What I tried | What Claude voice did | What the competition did |
| --- | --- | --- |
| Shared two MP3s (a song, a podcast pilot) | Saw filenames only; no transcription, no playback | Gemini transcribes audio natively and returns lyrics or speaker breakdown |
| Asked it to work on my Godot map on my PC | No filesystem, shell or desktop connector in the voice surface; pointed me to Claude Code CLI | Grok and Meta assistants connect to the desktop from voice |
| Wanted a code review → re-review → debug → PR loop by voice | Not available outside Claude Code | Grok/Meta ran the full loop, switching architect, backend and reviewer roles |
| Pasted a message about Fable 5 into Gemini | n/a | Gemini locked onto the Fable video game and never searched |

The last row is the mirror image of the first three: a competitor failing where Claude did not. It is in the table because it motivates the first feature ask below. The reasoning half of Claude is what built my confidence in AI; a lost Fable 5 Godot build (roughly $300 of API spend, GTA-feel city map, lost with a Hermes session) is the proof I wish I had kept. The hands and ears are what is missing.

## Headline: a conversation, not a syntax exam

The strongest ask is voice that resolves vague references to concrete code, then acts on it. That is how a senior engineer works with a designer, and it is a legitimate feature, not a nice-to-have.

I started programming at 11 and only ever cared about game languages. At 43 the only language I care about is telling AI what to learn. I cannot always pronounce "sycophancy", and in a C# session I will say "the correlation phase, the table thing I can flip down" instead of naming the construct. The right behaviour is a guess-confirm loop:

1. I describe the thing by its shape.
2. The model offers two or three candidates (a dictionary lookup? a foreach over a collection? a switch on type?).
3. I say "yeah, that one."
4. The model opens the actual code and checks whether that is why the In the Red district join breaks between two ends of the city.

Every confirmed guess is also training signal: a mapping from how builders talk to what the code is called. The feature improves the model while it serves the user.

## The feature asks

Eight asks, each a product or permissions decision rather than new research. The first and the last are the ones to lead with.

| # | Ask | Framing for Anthropic |
| --- | --- | --- |
| 1 | Resolve vague references to concrete code by voice (guess → confirm → inspect → act) | The headline above. Senior-engineer-with-a-designer behaviour. |
| 2 | Search immediately on a fresh chat for any unfamiliar proper noun or version number, and keep searching across the first five turns to pin down what the user means | Confidence suppresses the search call. Treat "Fable 5" as a signal to verify, not a familiar word. |
| 3 | Let voice use a connector the user already owns (I reach my PC over Tailscale) | A permissions decision, not an engineering one. The tunnel exists. |
| 4 | Give the voice agent its own sandbox (a VM or container) so it can browse and run things from a voice session | Cowork and computer-use already do this; expose it to voice. |
| 5 | Listen to shared audio: transcription, speaker breakdown, lyrics | Gemini has done this natively for a while. |
| 6 | MCP assessment by voice, visibility: "what can you reach right now?" answered plainly | Tonight this only happened because I pushed. It should be first-class. |
| 7 | MCP control by voice: "connect my Tailscale box", "add the Godot server", with the model confirming what the server exposes before use | Replaces the JSON-config step, which is the same CLI friction voice is meant to remove. |
| 8 | The full agentic loop from voice: read repo → review → re-review → debug → PR, switching architect, backend and reviewer modes | Claude Code does the loop; typing into a CLI breaks the voice workflow. Kimi's OK Computer mode and Agent Swarm show the walk-away version is shipping. |

## The protocol stack today

Three open layers exist, and all three move messages. None of them moves understanding. When Claude hands Kimi a task, Kimi gets text and rebuilds the whole mental model from scratch, and whatever it learns dies with the session.

| Layer | What it does | Who is behind it | What it does not carry |
| --- | --- | --- | --- |
| MCP (Model Context Protocol) | One model reaching tools and data | Anthropic, open-sourced | Agent-to-agent handoff |
| A2A (Agent2Agent) | One agent handing a task to another, discovery, coordination | Google, 2025; v1 in 2026 | Sub-agent or tool calls; shared working state |
| ACP (Agent Communication Protocol) | Cross-platform discovery, transport and semantic alignment; pitched as the "TCP/IP of the agentic web", federated rather than brokered | 2026 research paper | Any learning from the exchange |
| ANP (Agent Network Protocol) | Agents finding each other across the open internet | Emerging | State; learning |

The self-training piece people ask about is not in any of these. They are pure comms layers. The gap between them and a learning layer is where the walls are going up between labs, and it is where the rest of this document lives.

Sources: [A2A Protocol](https://a2a-protocol.org), [Google A2A announcement](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/), [IBM on A2A](https://www.ibm.com/think/topics/agent2agent-protocol), [Kimi on Wikipedia](<https://en.wikipedia.org/wiki/Kimi_(AI)>). The ACP paper is "Beyond Context Sharing: A Unified Agent Communication Protocol (ACP) for Secure, Federated, and Autonomous Agent-to-Agent Orchestration" (2026); link to be added from Dino's own research notes.

## ARFP: the state-transfer layer

ARFP (Adaptive Reconstruction Fabric Protocol, profile arfp-exact-delta-v1) is Dino Silvestri's state-transfer layer. It is deterministic, not neural: working state is externalized as canonical JSON (RFC 8785), every object is SHA-256 content-addressed, and what crosses the wire is an exact worktree delta against a pinned baseline hash plus a dependency DAG. The receiver reconstructs bit-exact, hash-verified state. Any model that can parse structured state and apply deltas can participate, so cross-architecture translation is solved at the protocol boundary, not inside the weights.

- **Adaptive**: not lossy fidelity. Adaptivity is scoped DAG slicing (a fast worker pulls only the leaf nodes, interface contracts and summary hashes its task needs; a reviewer pulls the wider upstream graph) and plan equifinality (different agents may decompose a task differently so long as their deltas converge on a state that satisfies the contract and passes conflict checks).
- **Reconstruction**: exact and deterministic. A lossy unpack would drift from the canonical baseline and fail on merge, so the protocol never allows it.
- **Fabric**: sits beside MCP, A2A and ACP as the thing they hand off; it does not replace them.
- **Six-object spine**: Target (pinned baseline hash) → Contract (permitted mutation scope and invariants) → Plan (dependency DAG) → Worktree Payload (untrusted delta, carried in OFFER / RESULT envelopes) → Validation Approval (hash, schema, conflict and boundary checks) → Commit Receipt (signed sign-off that advances the head). State never advances by default.

The "trains off itself" loop is verification, not compression. Every accepted or rejected RESULT yields a hash-labelled (Contract, Plan DAG, Delta, Validation) tuple; when one specialist fails validation and a second succeeds on the same contract, the control plane captures a contrastive pair grounded in deterministic verification (RLVR / DPO). Training the router and planner on receipt metadata, not worktree content, keeps competitor tokens out of anyone's weights and inside each lab's terms of service.

Status: public repo in progress under Dino's GitHub (October 2026); architecture pieces and handoff docs exist. Internals are Dino's and are deliberately not detailed here.

## Federation: specialists under a control plane

Cheap state transfer is what makes federation possible. Once Claude and Kimi can exchange working state through ARFP, each runs as a specialist and a control plane above them schedules work, routes it to whoever is strongest, and merges the results.

&#91;embedded content: federated specialists under one control plane · 4 layers\]

The analogy is Kubernetes for cognition: the models are interchangeable workers, the orchestrator owns the user, and the merged layer in the middle gets richer with every exchange. This is what the multi-agent orchestration crowd is already building, but with one vendor's models today instead of across labs.

## Why this matters to Anthropic

Whoever owns the control plane owns the user relationship, and the individual models become interchangeable workers underneath it. That is the strategic reason to care about the voice gaps above: the orchestrator the user talks to is the control plane, and if a competitor's voice assistant is the thing that reaches the desktop, hears the audio, and dispatches the work, Claude becomes one of the workers it calls.

Three consequences follow.

1. The eight feature asks are not polish. They decide which product sits at the top of the stack.
2. Open protocols are the right bet, and Anthropic made it with MCP. The learning layer above the protocols is where closed walls are going up, and it is too early in the design phase for labs to cut each other off there; it slows the whole field down.
3. A state-transfer layer is the piece nobody has standardised. Whoever defines it defines how models cooperate, including across the export-control line that pulled Fable 5 for three weeks in June 2026.

This section is analysis, not something looked up. It is labelled as such so it is not quoted as fact.

## Open problems and honest pushback

The idea is strong enough to survive these; a reader at Anthropic will raise them, so they go in the document first.

| Problem | Why it is hard | What would answer it |
| --- | --- | --- |
| Cross-architecture reconstruction (resolved by the spec) | Would be fatal if ARFP shipped latents. It ships canonical JSON deltas and hashes, so no translator is needed | Already answered by arfp-exact-delta-v1; a two-model demo applying the same delta would make it visible |
| Data provenance in the merged layer | Competing labs, live export controls; who owns what a handoff teaches | A provenance tag on every reconstructed payload; opt-in per lab |
| Security surface | A worktree delta is a high-bandwidth vector for indirect injection if the receiver treats it as more than data | The spec's Contract / Validation Approval / Commit Receipt split: payloads cannot widen their own contract or bypass validation. Remaining gaps are pre-OFFER semantic classification (hashes are content-blind) and path-topology leakage, closed by policy-as-code labels and canonical path aliasing |
| Commercial and political blockers | Moonshot and Anthropic compete; a sanctioned US-to-Chinese-lab integration is unlikely to clear Commerce today | Lead with federation between US labs or open-weight models; keep the cross-border case as the long-run argument |
| "Senior-level" is a feeling | A voice review loop can feel senior while shipping subtle bugs | Judge the competitors' loops by whether their PRs hold up a week later, and say so in the email |

One wording fix for the email: the central layer is federated specialists under an orchestrator, not Kubernetes. Kubernetes schedules identical containers; the analogy only holds for the control plane, not for the knowledge pooling. A sharp reader will catch it if the two are conflated.

## Next steps

Two outputs come from this document: an email to Anthropic and a LinkedIn post. The email carries the feature asks and the strategic argument; the post carries the headline with the C# story beside it.

- [ ] Cut the email to one page: the gap table, the eight asks, the control-plane argument, a link to this doc for the rest
- [ ] Find the right recipient (product or developer-relations, not the generic feedback form) and send through the in-app feedback button as well
- [ ] Draft the LinkedIn post: lead with the "correlation phase, the table thing I flip down" example, land on "a conversation, not a syntax exam"
- [ ] Add the ACP paper link from Dino's research notes to the Sources line above
- [ ] Rerun the Godot GTA-feel map prompt on Fable 5.1 and on Gemini side by side, commit both outputs to a repo, and attach the comparison if it supports the quality claim
- [ ] Decide how much of ARFP to disclose publicly before the repo is ready; this doc keeps internals out on purpose
