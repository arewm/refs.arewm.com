# Related Projects

Projects operating in adjacent problem spaces. These are not dependencies or
endorsed integrations — they solve different problems at different layers.

## Attribution

### [Agent Trace](https://agent-trace.dev)

An open spec for recording per-file and per-line attribution of AI-generated
code (human, AI, mixed, or unknown). Agent Trace answers "who wrote this
line?" while the agent-commit buildType answers "how was this commit
produced?"

Agent Trace is most valuable in interactive workflows where humans and agents
co-edit files. For fully headless agent runs — the primary use case for the
agent-commit buildType — attribution is trivially "the agent wrote everything"
and the interesting questions shift to provenance: sandbox policy, runtime
observations, model identity.

The two specs are complementary. A consumer could use agent-trace for
code-level attribution and the agent-commit buildType for supply-chain-level
provenance of the same commit.

## Sandbox Enforcement and Signing

### [nono](https://nono.sh)

An agent sandbox platform with OS-level enforcement (macOS Seatbelt, Linux
Landlock), a built-in observation layer, and two attestation capabilities:

- **Instruction file signing** — attests that a key holder approved the
  content of agent instruction files (e.g., `SKILLS.md`). Supports keyed
  (ECDSA P-256, system keystore) and keyless (Fulcio/Rekor) signing via
  Sigstore bundles. Predicate types: `instruction-file/v1`,
  `trust-policy/v1`, `multi-file/v1`.

- **Audit session attestation** — signs a session-scoped DSSE/in-toto
  attestation binding a session ID, timing, command, and the Merkle root
  of an append-only audit event log. The attestation subject is
  `audit-session:{session_id}` with the event log Merkle root as the
  digest — it proves log integrity for a session, not provenance of a
  specific output artifact.

**Observation model.** nono observes sessions through its own mediation
layers rather than external instrumentation like eBPF or ptrace:
capability requests over a supervisor IPC socket (with peer PID
extraction), L7 network traffic through its built-in proxy (method, path,
status, auth mechanism), and periodic Merkle-tree filesystem snapshots of
tracked paths. This provides good coverage of mediated activity but does
not capture syscall-level detail — file reads within already-granted
access, for example, are not individually logged. No OTel integration is
required or used.

**Subagent visibility.** nono treats the entire sandboxed process tree as
a single unit. Child processes inherit the parent's Landlock/Seatbelt
sandbox (they cannot be restricted further), share one session ID, and
are indistinguishable in the network audit log (the proxy sees
`127.0.0.1` with no PID attribution). The buildType's `invocationId`-based
subagent correlation — where a coordinator and implementer have separate
observation streams — has no equivalent in nono's model.

**Relationship to the buildType.** nono's signing infrastructure (DSSE +
in-toto + Sigstore) is the kind of envelope a builder platform would use
to sign an agent-commit provenance predicate. Its audit data — network
events, capability decisions, filesystem snapshots — could serve as input
for populating the buildType's `observed-*` byproducts, though the
observation granularity differs (mediated-access logging vs. the
syscall-level continuous observation the buildType's schema assumes). The
core gap remains the provenance schema: nono proves that an audit log for
a session is authentic; the buildType describes how a specific commit was
produced, under what constraints, with what observations, and with
per-subagent attribution.

## Sandbox Platforms

### [OpenShell](https://github.com/NVIDIA/OpenShell)

NVIDIA's open-source agent sandbox platform. Provides an L7 proxy with
OPA-based network policy, optional Landlock filesystem enforcement, and
seccomp syscall filtering. Observability captures network connections
(OCSF class 4001/4002) and process lifecycle (class 1007) events but has no comprehensive filesystem access observation (only
Landlock policy enforcement events, not per-file read/write auditing) —
a gap the buildType's `observed-filesystem` byproduct addresses.

A [proposed sidecar architecture](https://github.com/NVIDIA/OpenShell/issues/981)
separates the supervisor proxy and agent into distinct containers with
separate cgroups, which would simplify external eBPF-based observation by
eliminating the need to distinguish supervisor from agent activity within
a shared cgroup. In this model, the proxy sidecar populates
`observed-network` byproducts; an external eBPF observer attached to the
agent container's cgroup populates `observed-filesystem` and
`observed-process`.

## Attestation Tooling

Projects in this section use signing infrastructure from
[Sigstore](https://sigstore.dev) (keyless, OIDC-bound) or
[SPIFFE/SPIRE](https://spiffe.io) (workload identity). These are not
described separately as they are envelope-level concerns, not
attestation-content concerns.

### [witness](https://github.com/in-toto/witness)

An in-toto attestation generator that wraps command execution. Captures
pre/post filesystem state (material/product attestors), command execution
(args, exit code, stdout/stderr), and optionally all subprocesses and
file access via ptrace (Linux-only). Supports Sigstore keyless signing,
SPIFFE/SPIRE workload identity, and KMS backends.

Wrapping an agent session (`witness run -- agent-command`) produces a
signed record of filesystem changes and process execution. The
buildType's runtime observation byproducts capture similar data with
timestamps and network activity that witness does not cover. The two
approaches are complementary: witness provides per-step signing and
supply chain policy verification; eBPF-based observation provides
continuous, timestamped ground truth including network activity.
Per-action witness attestations can be referenced as byproducts in the
buildType via the v0.3 referenced attestations mechanism.

### [aflock](https://github.com/aflock-ai/aflock)

A policy enforcement and attestation system for AI agents, built on
SPIRE's workload identity model and go-witness. Runs as an MCP server
that derives a cryptographic agent identity from process introspection
(PID, model, tools, environment, policy digest) and signs per-action
in-toto attestations using a key the agent never sees.

aflock's `.aflock` policy file defines constraints (tool restrictions,
file access patterns, spend limits, domain allowlists) with configurable
enforcement modes (fail-fast or post-hoc). Its sublayout mechanism
supports sub-agent delegation with stricter-than-parent constraints and
identity chaining.

aflock operates at the MCP layer — it sees what the agent asks MCP tools
to do, not what the agent does via direct syscalls. External observation
(e.g., eBPF) captures activity that bypasses MCP. The two are
complementary: aflock's per-action attestations can be referenced as
byproducts in the buildType via the v0.3 referenced attestations
mechanism, and a verifier can cross-reference aflock's claims against
external observations.

## Agent Observability

### [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)

Emerging OTel semantic conventions (`gen_ai.*`) define span attributes
for AI operations including model identity, token usage, and agent tool
calls. Multiple projects bridge agent runtime hooks to OTel backends
(e.g., claude-code-otel, Pydantic logfire plugin, Vercel AI SDK
telemetry). These provide operational observability — latency, cost,
tool call sequences — but do not produce supply chain attestations.

The buildType uses supply chain attestation formats (in-toto/SLSA) to
capture data that overlaps with what OTel traces record (tool calls,
timing, model identity), but in a signed, verifiable, policy-evaluable
document rather than an operational trace.

A builder platform could consume OTel spans as an input source for
populating buildType fields. Model identity attributes (`gen_ai.*`) map
to `internalParameters.agent.model`. Tool call spans with agent/subagent
identifiers map to `internalParameters.subagents[]` entries. Span
timestamps enable correlation between semantic events (tool calls) and
runtime observations (process executions, file writes) via matching
`invocationId` fields. OTel provides the attribution context — which
tool call triggered which action — while external observation (e.g.,
eBPF) provides the ground truth that the action actually occurred.

In a sandboxed environment, OTel spans could be written to a file by
hook scripts and read by the builder platform after the run completes,
or exported via OTLP to a collector on the sandbox's internal network.
Because these spans originate inside the sandbox, they are untrusted —
a compromised agent could fabricate them. The builder platform should
corroborate OTel-reported events against external observations before
incorporating them into the attestation.

### [ToolHive](https://github.com/stacklok/toolhive)

MCP server management platform by Stacklok that verifies MCP server
image provenance via Sigstore before deployment. Attestation is at the
image level — proving the server binary is authentic — not at the
invocation level. The buildType captures MCP tools as resolved
dependencies with digests, which complements ToolHive's image
verification: ToolHive ensures the tool is authentic, the buildType
records what the tool did during the agent run.