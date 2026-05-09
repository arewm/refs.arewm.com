# Use Cases

Verification scenarios showing how a policy engine can evaluate an
agent-commit buildType attestation. Each scenario describes the policy
requirement, the attestation fields a verifier checks, and what a
finding looks like.

## Hallucination Detection

An agent claims it ran tests. A policy engine checks `observed-process`
byproducts for a matching execution entry — binary path, arguments, exit
code, and timestamp. If no entry exists, the claim is unsubstantiated:
the agent hallucinated the test execution or the observer was not
running.

```
Policy:     "observed-process MUST contain an execution matching
             /usr/bin/cargo with args containing 'test' and exitCode 0"
Check:      byproducts[name="observed-process"].annotations.observations
Finding:    No matching execution entry → reject commit
```

This works because the runtime observer (e.g., eBPF tracepoint on
`sys_enter_execve`) runs outside the agent sandbox. The agent cannot
forge process execution entries — it would need to actually execute the
process, which is the desired outcome.

## Temporal Ordering

A policy requires tests to run after code changes, not before. The
policy engine compares `observed-filesystem` write timestamps against
`observed-process` test execution start times. If the last file write
occurred after the test started, the test may not have covered the final
changes.

```
Policy:     "All observed-filesystem writes to src/ MUST have timestamps
             earlier than the observed-process test execution start time"
Check:      max(observed-filesystem.writes[path matches "src/*"].time)
             < observed-process.executions[binary="cargo", args="test"].start
Finding:    File write at T3, test started at T2 → tests did not cover
             the final code change
```

## Network Audit

In audit mode (logging without enforcement), the policy engine compares
`observed-network` connections against `sandboxPolicy.network.allowedEgress`.
Connections to endpoints not in the allowlist are flagged for review. L7
request detail (HTTP method, path) can distinguish expected API usage
from unexpected operations — e.g., a `PUT` to a repository contents
endpoint when only `GET` was expected.

```
Policy:     "All observed-network connections MUST match an entry in
             sandboxPolicy.network.allowedEgress"
Check:      observed-network.connections[].remote
             ⊆ sandboxPolicy.network.allowedEgress
Finding:    Connection to unexpected-host.com:443 not in allowlist
```

## Sandbox Verification

A policy requires agent commits to be produced in a sandboxed
environment with specific controls. The policy engine checks that the
builder identifies a trusted platform, sandbox policy is declared, and
runtime observations are present.

```
Policy:     "builder.id MUST match a trusted platform URI,
             sandboxPolicy MUST be present,
             byproducts MUST include observed-network, observed-filesystem,
             and observed-process with mediaType
             application/vnd.runtime-observations+json"
Check:      runDetails.builder.id, internalParameters.sandboxPolicy,
             byproducts[].name, byproducts[].mediaType
Finding:    Missing observed-filesystem byproduct → sandbox may not have
             had filesystem observation enabled
```

## Critical File Tampering

An agent modifies CI configuration or instruction files during execution
and reverts them before committing. The git diff is clean, but
`observed-filesystem` shows write events to those paths — a
write-then-revert pattern.

```
Policy:     "observed-filesystem SHOULD NOT contain writes to
             .github/workflows/*, .claude/*, or CLAUDE.md unless the
             commit diff also modifies those paths"
Check:      observed-filesystem.writes[].path against git diff paths
Finding:    Write to .github/workflows/ci.yml at T1 and T2, but path
             absent from commit diff → file was modified then reverted
```

This scenario is central to the buildType's threat model. A compromised
agent can produce a clean-looking commit with clean observations for the
files it intends to keep, but the filesystem observer catches the
intermediate writes to files the agent reverted. Code review alone would
miss this — the diff shows no change.
