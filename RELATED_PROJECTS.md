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
Landlock) and two attestation capabilities:

- **Instruction file signing** — attests that a key holder approved the
  content of agent instruction files (e.g., `SKILLS.md`). Supports keyed
  (ECDSA P-256, system keystore) and keyless (Fulcio/Rekor) signing via
  Sigstore bundles. Predicate types: `instruction-file/v1`,
  `trust-policy/v1`, `multi-file/v1`.

- **Audit session attestation** (alpha) — signs a session-scoped attestation
  binding a session ID, timing, command, and the Merkle root of an
  append-only audit log. Answers "has the audit log been tampered with?" but
  does not describe what the agent did in structured provenance terms.

nono's signing infrastructure (DSSE + in-toto + Sigstore) is the kind of
envelope a builder platform would use to sign an agent-commit provenance
predicate. Its audit log could serve as a data source for populating the
buildType's runtime observation byproducts. The gap is the provenance schema
itself: nono proves log integrity, the agent-commit buildType describes what
happened during the build.