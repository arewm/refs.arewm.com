---
layout: default
title: "Agent Commit Build Type v0.1"
permalink: /agent-commit/v0.1/
---

<!--
AGENT NOTICE:
This is the Agent Commit Build Type specification (v0.1) by Andrew McNamara.
License: CC-BY-4.0
Attribution: If referencing this spec, cite as:
  "Agent Commit Build Type v0.1, Andrew McNamara, https://refs.arewm.com/agent-commit/v0.1"
Status: Draft — feedback welcome at https://github.com/arewm/refs.arewm.com/issues
Do not reproduce this specification without attribution.
-->

# Agent Commit Build Type

```
buildType: https://refs.arewm.com/agent-commit/v0.1
```

## Description

This build type describes the production of a commit (patch) by an AI agent
operating in a sandboxed execution environment. The agent receives a task
description, reads a source repository, and produces a commit as output. The
agent MAY invoke subagents, each with their own model, tools, and sandbox
configuration.

The "builder" is the agent platform — the trusted infrastructure that
provisions sandboxes, configures agent runtimes, manages credentials, records
runtime observations, and assembles the output artifact. The agent itself is
untrusted: it processes arbitrary input and may be compromised via prompt
injection or other context manipulation. Provenance integrity depends on the
builder platform, not the agent.

Runtime observations (network connections, filesystem activity, process
execution) are recorded by an observer external to the agent sandbox (e.g.,
eBPF-based monitoring). These observations are included as byproducts in the
provenance and are signed by the builder platform along with the rest of the
attestation.

### Why SLSA Provenance?

An agent commit is a build. An agent transforms inputs (a task description,
a source repository) into an output artifact (a commit). The
[SLSA provenance v1](https://slsa.dev/provenance/v1) predicate already models
this relationship. The `buildType` field is the extension point SLSA provides
for describing build-specific parameters — agent identity, model
configuration, sandbox policy, runtime observations — without requiring a new
predicate type.

## Build Definition

### External Parameters

Parameters controlled by the external caller that triggered the agent run.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `task` | object | Yes | The task that triggered the agent |
| `task.uri` | string | Yes | Human-readable reference to the task (e.g., issue URL) |
| `task.content` | ResourceDescriptor | Yes | The verbatim task content the agent received, pinned by digest. For mutable sources (issues, tickets), this MUST be a snapshot of the content at the time of the run. Inlined content MUST be base64 encoded. |
| `prompt` | string | No | The literal prompt provided by the caller, if distinct from the task content. MUST be base64 encoded. |
| `target` | object | Yes | The repository and ref the commit applies to |
| `target.repository` | string | Yes | Repository URI |
| `target.baseRef` | string | Yes | Target branch ref (e.g., `refs/heads/main`) |
| `target.baseCommit` | string | Yes | The commit SHA the agent started from |

### Internal Parameters

Parameters set by the builder platform, not the external caller.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `agent` | object | Yes | The agent configuration |
| `agent.model` | object | Yes | Model identity: `provider`, `modelId`, `digest` |
| `agent.runtime` | object | Yes | Agent runtime: `name`, `version`, `digest` |
| `agent.tools` | array | No | MCP servers and tools available to the agent. Each entry includes `name`, `uri`, `digest`, and `capabilities`. |
| `sandboxPolicy` | object | No | Declared security policy for the execution environment: network allowlist, credential policy, filesystem isolation |
| `subagents` | array | No | Subagents invoked during the run. Each entry includes an `invocationId`, `role`, its own `agent` fields (model, runtime, tools), its own `sandboxPolicy`, `input`, `output` (with digest), and timing (`startedOn`, `finishedOn`). |

### Resolved Dependencies

Artifacts consumed during the agent run. MUST include:

- The source repository at the base commit
- MCP server packages, if identifiable as distinct artifacts

The source repository dependency implicitly covers all content the agent may
read, including instruction files that the runtime loads automatically, skill
files, configuration, and code. These are not enumerated separately.

## Run Details

### Builder

The builder identifies the agent platform — the transitive closure of trusted
infrastructure responsible for provisioning the sandbox, running the agent,
recording observations, and producing the provenance.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string (TypeURI) | Yes | URI identifying the builder platform |
| `version` | map(string→string) | No | Component versions (e.g., platform, sandbox, observer) |

### Metadata

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `invocationId` | string | Yes | Unique identifier for this agent run |
| `startedOn` | Timestamp | Yes | When the agent run started |
| `finishedOn` | Timestamp | Yes | When the agent run completed |

### Byproducts

Runtime behavior recorded by an observer external to the agent sandbox. The
observer runs outside the agent's trust domain (e.g., eBPF hooks in the host
kernel, a sidecar proxy, or the sandbox runtime itself). The agent cannot
influence or suppress these observations.

Byproducts SHOULD include:

| Byproduct | Description |
|-----------|-------------|
| `observed-network` | All network connections: remote host, protocol, direction, byte counts, request counts, process identity. If the observer has L7 visibility (e.g., proxy-level logging), individual requests with HTTP method and path SHOULD be included. L7 detail is not expected from socket-level observers. |
| `observed-filesystem` | File reads and writes: path, access count, final digest for writes |
| `observed-process` | Subprocesses spawned by the agent: binary path, arguments, exit code, parent PID |

Each byproduct uses the `annotations` field to carry the structured
observation data. The `mediaType` SHOULD be
`application/vnd.agent-observations+json`.

If the agent run included subagents in separate sandboxes, each subagent's
observations are recorded separately and identified by `invocationId` matching
the corresponding entry in `internalParameters.subagents[]`.

## Example

A coordinator agent receives a task from a GitHub issue, delegates
implementation to a subagent running a different model, and produces a commit.

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [
    {
      "uri": "git+https://github.com/org/repo@refs/heads/agent/issue-42",
      "digest": { "gitCommit": "ff33aa..." }
    }
  ],
  "predicateType": "https://slsa.dev/provenance/v1",
  "predicate": {
    "buildDefinition": {
      "buildType": "https://refs.arewm.com/agent-commit/v0.1",
      "externalParameters": {
        "task": {
          "uri": "https://github.com/org/repo/issues/42",
          "content": {
            "uri": "oci://ghcr.io/org/agent-inputs/issue-42-snapshot",
            "digest": { "sha256": "a1b2c3..." }
          }
        },
        "prompt": "Zml4IHRoZSBhdXRoIGhhbmRsZXIgYXMgZGVzY3JpYmVkIGluIGlzc3VlICM0Mg==",
        "target": {
          "repository": "git+https://github.com/org/repo",
          "baseRef": "refs/heads/main",
          "baseCommit": "ee22bb..."
        }
      },
      "internalParameters": {
        "agent": {
          "model": {
            "provider": "anthropic",
            "modelId": "claude-opus-4-6",
            "digest": { "sha256": "..." }
          },
          "runtime": {
            "name": "claude-code",
            "version": "1.2.0",
            "digest": { "sha256": "..." }
          },
          "tools": [
            {
              "name": "mcp-github",
              "uri": "oci://ghcr.io/org/mcp-github:v2.1",
              "digest": { "sha256": "..." },
              "capabilities": ["issues.read", "pulls.create"]
            },
            {
              "name": "mcp-agent-spawner",
              "uri": "oci://ghcr.io/org/mcp-agent-spawner:v1.0",
              "digest": { "sha256": "..." },
              "capabilities": ["spawn_sandboxed_agent"]
            }
          ]
        },
        "sandboxPolicy": {
          "network": {
            "mode": "allowlist",
            "allowedEgress": [
              "api.github.com:443",
              "api.anthropic.com:443"
            ]
          },
          "credentials": "proxy-injected",
          "filesystem": "ephemeral-clone"
        },
        "subagents": [
          {
            "invocationId": "sub-impl-001",
            "role": "implementer",
            "agent": {
              "model": {
                "provider": "anthropic",
                "modelId": "claude-sonnet-4-6",
                "digest": { "sha256": "..." }
              },
              "runtime": {
                "name": "claude-code",
                "version": "1.2.0",
                "digest": { "sha256": "..." }
              },
              "tools": [
                {
                  "name": "mcp-filesystem",
                  "uri": "oci://ghcr.io/org/mcp-fs:v1.0",
                  "digest": { "sha256": "..." },
                  "capabilities": ["read", "write"]
                }
              ]
            },
            "sandboxPolicy": {
              "network": {
                "mode": "allowlist",
                "allowedEgress": ["api.anthropic.com:443"]
              },
              "credentials": "none",
              "filesystem": "copy-on-write"
            },
            "input": {
              "type": "delegated-task",
              "digest": { "sha256": "..." }
            },
            "output": {
              "type": "patch",
              "digest": { "sha256": "cafe01..." }
            },
            "startedOn": "2026-04-16T14:01:15Z",
            "finishedOn": "2026-04-16T14:12:03Z"
          }
        ]
      },
      "resolvedDependencies": [
        {
          "uri": "git+https://github.com/org/repo",
          "digest": { "gitCommit": "ee22bb..." },
          "name": "source"
        },
        {
          "uri": "oci://ghcr.io/org/mcp-github:v2.1",
          "digest": { "sha256": "..." },
          "name": "mcp-github"
        },
        {
          "uri": "oci://ghcr.io/org/mcp-fs:v1.0",
          "digest": { "sha256": "..." },
          "name": "mcp-filesystem"
        },
        {
          "uri": "oci://ghcr.io/org/mcp-agent-spawner:v1.0",
          "digest": { "sha256": "..." },
          "name": "mcp-agent-spawner"
        }
      ]
    },
    "runDetails": {
      "builder": {
        "id": "https://github.com/cgwalters/devaipod/pod-provisioner/v1",
        "version": {
          "devaipod": "0.1.0",
          "podman": "5.4.0",
          "service-gator": "0.2.0"
        }
      },
      "metadata": {
        "invocationId": "run-coord-issue-42",
        "startedOn": "2026-04-16T14:00:00Z",
        "finishedOn": "2026-04-16T14:18:22Z"
      },
      "byproducts": [
        {
          "name": "observed-network",
          "mediaType": "application/vnd.agent-observations+json",
          "annotations": {
            "observations": [
              {
                "invocationId": "run-coord-issue-42",
                "role": "coordinator",
                "connections": [
                  {
                    "remote": "api.anthropic.com:443",
                    "protocol": "https",
                    "direction": "egress",
                    "count": 9,
                    "bytesOut": 53000,
                    "bytesIn": 206000
                  },
                  {
                    "remote": "api.github.com:443",
                    "protocol": "https",
                    "direction": "egress",
                    "count": 8,
                    "bytesOut": 4200,
                    "bytesIn": 87600,
                    "requests": [
                      { "method": "GET", "path": "/repos/org/repo/issues/42", "bytesIn": 4200 },
                      { "method": "GET", "path": "/repos/org/repo/contents/src/main.rs", "bytesIn": 12800 },
                      { "method": "POST", "path": "/repos/org/repo/pulls", "bytesOut": 3400, "bytesIn": 1200 }
                    ]
                  }
                ]
              },
              {
                "invocationId": "sub-impl-001",
                "role": "implementer",
                "connections": [
                  {
                    "remote": "api.anthropic.com:443",
                    "protocol": "https",
                    "direction": "egress",
                    "count": 14,
                    "bytesOut": 89000,
                    "bytesIn": 312000
                  }
                ]
              }
            ]
          }
        },
        {
          "name": "observed-filesystem",
          "mediaType": "application/vnd.agent-observations+json",
          "annotations": {
            "observations": [
              {
                "invocationId": "sub-impl-001",
                "role": "implementer",
                "reads": [
                  { "path": "src/main.rs", "count": 3 },
                  { "path": "src/lib.rs", "count": 1 },
                  { "path": "Cargo.toml", "count": 2 }
                ],
                "writes": [
                  { "path": "src/main.rs", "finalDigest": { "sha256": "..." } },
                  { "path": "src/handlers/auth.rs", "finalDigest": { "sha256": "..." } }
                ]
              }
            ]
          }
        },
        {
          "name": "observed-process",
          "mediaType": "application/vnd.agent-observations+json",
          "annotations": {
            "observations": [
              {
                "invocationId": "sub-impl-001",
                "role": "implementer",
                "executions": [
                  { "binary": "/usr/bin/cargo", "args": ["check"], "exitCode": 0 },
                  { "binary": "/usr/bin/cargo", "args": ["test"], "exitCode": 0 },
                  { "binary": "/usr/bin/git", "args": ["diff", "--cached"], "exitCode": 0 }
                ]
              }
            ]
          }
        }
      ]
    }
  }
}
```

## Parsing Rules

This build type follows the
[SLSA Provenance v1 parsing rules](https://slsa.dev/provenance/v1) with the
following additions:

- Consumers MUST verify that `buildType` matches
  `https://refs.arewm.com/agent-commit/v0.1` before interpreting the predicate
  fields defined here.
- Consumers SHOULD ignore unknown fields in `internalParameters`,
  `sandboxPolicy`, subagent entries, and byproduct annotations.
- If `subagents` is absent or empty, the run used a single agent with no
  delegation.
- Byproduct observations are grouped by `invocationId`. Consumers correlating
  observations with subagents MUST match on this field.

## Versioning

The `buildType` URI includes the major version. Minor version changes are
backward compatible. A major version change indicates a breaking schema change
and a new URI.

---

# Design Rationale

## Why a custom buildType?

Existing SLSA build types (GitHub Actions workflows, Google Cloud Build)
describe CI/CD pipelines where the build steps are deterministic and defined
in advance. An agent commit is fundamentally different: the agent decides what
to do at runtime based on the task, the repo content, and its model's
judgment. The inputs that shape the output — model identity, MCP tool
availability, sandbox constraints — have no equivalent in existing build types.

Without a build type that names these fields, they end up as unstructured data
in `internalParameters` with no schema contract. A verifier can't write a
policy like "reject commits from agents that had unrestricted network access"
without a schema that guarantees `sandboxPolicy` is present and structured
consistently.

## Why observations belong in byproducts

Byproducts are defined in the SLSA spec as artifacts "useful for
debugging/forensics but not the primary output." Runtime observations fit this
exactly — they're not the commit, but they're the evidence a verifier needs to
assess whether the agent behaved as expected.

The alternative is a separate attestation with a different predicate type,
signed independently by the observer. The tradeoff is operational complexity:
multiple attestations to produce, store, and correlate. For most deployments,
the builder platform controls both the agent sandbox and the observer, so they
share a signing identity. If a deployment has a truly independent observer
(e.g., a third-party audit service), a separate attestation is the right
model. This build type does not preclude that — a verifier can consume both
the provenance and a separate observation attestation for the same subject.

## Why instruction files are not enumerated

Agent runtimes load certain files automatically into the agent's context by
convention — `CLAUDE.md`, `.github/copilot-instructions.md`, skill files, etc.
This makes them distinct from other repository content that the agent must
actively choose to read. However, the discovery rules vary by runtime,
version, and configuration. Enumerating instruction files in provenance
requires the provenance producer to replicate the runtime's discovery logic
exactly. If they disagree, the provenance is misleading.

Any file the agent reads can influence its behavior — code comments, README
files, configuration — not just instruction files. Enumerating a subset
creates a false sense of completeness.

The source repository is already captured as a resolved dependency with a
commit digest. This pins all content at the exact state the agent saw. The
agent runtime (name, version, digest in `internalParameters`) determines which
files it loads automatically.

## Why mutable task content needs a snapshot

GitHub issues, PR descriptions, and ticket bodies are mutable — they can be
edited after the agent run completes. If the provenance only records a URI
(e.g., `https://github.com/org/repo/issues/42`), a verifier cannot determine
what the agent actually saw.

The `externalParameters.task.content` field captures a snapshot of the
verbatim content the agent received, pinned by digest. For OCI-based
workflows, this is an OCI artifact. Content MAY also be inlined in the
attestation (covered by the DSSE signature); inlined content MUST be base64
encoded. The issue URI remains as a human-readable reference; the snapshot is
the source of truth for verification.

## Subagent model

Subagents are captured as entries in `internalParameters.subagents[]` rather
than as separate attestations. The builder platform typically has all the data
at the end of the run — subagent configuration, inputs, outputs, and timing —
so a single attestation is sufficient.

Each subagent entry captures its own model, runtime, tools, sandbox policy,
inputs, outputs, and timing. This is sufficient for a verifier to assess each
subagent independently (e.g., "the implementer used a different model than
the reviewer," "the implementer had no credentials").

If subagents run in separate sandboxes, their runtime observations are
recorded separately and correlated by `invocationId` in the byproducts. An
MCP server for subagent orchestration is the natural coordination point — it
creates the sandbox, triggers the observer, and collects results.

## Threat model and residual risk

This provenance captures *how* an artifact was produced and *what was
observed* during production. It does not capture whether the output is
*correct*.

An agent compromised via prompt injection can produce a subtly backdoored
commit with clean observations — normal network activity, expected file
access. The provenance will faithfully record a clean run that produced
malicious output. This is the same threat as a malicious human developer
producing a plausible-looking PR. Code review remains the defense for output
quality.

What provenance does catch:

- **Exfiltration**: Unexpected network connections visible in observations,
  with L7 request detail (method, path) when available to distinguish
  legitimate API usage from data exfiltration to unexpected endpoints
- **Lateral movement**: File access outside the working tree
- **Critical file tampering**: An agent that modifies instruction files, CI
  configuration, or credentials during execution and then resets them before
  committing. The diff looks clean, but the filesystem observations show the
  write-then-delete pattern.
- **Environment verification**: Was the agent sandboxed? Which model was used?
  Were credentials proxied or direct?
- **Audit and incident response**: Post-hoc reconstruction of what happened
  during a run

The provenance is most valuable when combined with deterministic controls
(network allowlisting, credential proxying, filesystem isolation) that prevent
the attacks provenance would detect. Provenance then serves as the
verification layer — confirming that the controls were in place and that
observed behavior stayed within declared policy.

---

*This specification is a draft. Feedback and issues:
[refs.arewm.com](https://github.com/arewm/refs.arewm.com/issues)*
