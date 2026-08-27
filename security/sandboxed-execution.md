# Sandboxed Execution

An agent that can write or run code has crossed an important boundary: its output can consume compute, read files, make network requests, and change state. Security controls decide what the agent may request. A sandbox limits what code can reach when that request, the generated code, or an input is wrong.

This page builds on [Security for AI Systems](security-for-ai-systems.md). It focuses on the execution environment for generated or untrusted code.

## The Problem

Consider a data-analysis assistant. A user uploads a CSV and asks for a forecast. The assistant writes and runs a Python script. The script may loop forever, read a mounted credential file, send data to the internet, or exploit a runtime vulnerability.

The same risk exists when an agent runs code from a repository, notebook, browser automation script, or document. Prompt injection is one path for bad instructions, but a sandbox must also contain ordinary model mistakes and code bugs.

Running this work in the API server, a shared worker, or a developer laptop gives it far too much reach. Sandboxed execution gives each task a deliberately small workspace and makes the allowed reach explicit.

## Mental Model

Treat an execution sandbox like a disposable workshop: the worker gets only the tools, materials, time, and exits needed for one job. It does not receive the building keys just because it needs a workbench.

```text
Agent proposes code and inputs
              ↓
Policy creates a small execution contract
              ↓
Disposable sandbox runs the job
       ├── limited CPU, memory, and time
       ├── isolated files and identity
       └── restricted or no network
              ↓
Validated output crosses back to the application
```

The sandbox is a **blast-radius boundary**, not proof that the code is safe. It should be one layer in a larger design that also validates tool requests, authorizes access, and protects secrets.

## How It Works

1. **Classify the job.** Decide whether the agent needs code execution at all. Prefer a narrow application tool, such as `calculate_tax`, over an unrestricted shell whenever the work is predictable.
2. **Create an execution contract.** Define the allowed runtime, input files, writable output directory, CPU, memory, wall-clock deadline, process limit, and network destinations. Bind it to the user and task that requested the work.
3. **Start an isolated environment.** Create a fresh container with additional isolation, a microVM, or a capability-limited WebAssembly runtime. Do not reuse a dirty workspace from another user or task.
4. **Expose only needed resources.** Mount input files read-only. Give the job a new empty output directory. Use a short-lived, scoped credential only when it must call a specific service. Default to no network, then allow only named destinations if necessary.
5. **Run and enforce limits outside the code.** The host or sandbox runtime enforces time, memory, CPU, process, filesystem, and network limits. A prompt or a comment inside generated code is not a security control.
6. **Collect and validate the result.** Capture standard output, logs, exit status, and declared output files. Check output size, type, and schema before returning anything to the agent or storing it. Destroy the environment and revoke its credentials when the job ends.

## Important Concepts

### Isolation Boundary

The boundary separates the workload from the host, other tenants, and sensitive services. Native containers share the host kernel, so they are useful packaging and resource controls but should not automatically be treated as a sufficient boundary for hostile or high-risk code. Stronger choices can add an application kernel or hardware virtualization through a microVM.

The boundary follows the threat model. A trusted internal formatter may need ordinary process isolation. Arbitrary multi-tenant code usually deserves a stronger boundary.

### Capabilities and Least Privilege

Capabilities are specific things the job can do: read a directory, write a directory, connect to an API, or use a short-lived token. Start with none and add only what the job requires. A job with a strong VM boundary can still leak data if it receives a broad cloud credential or unrestricted network access. Isolation limits escape; capabilities limit what a contained workload can legitimately do.

### Ephemeral State

Each execution should start from a known image and empty working area. Inputs arrive through an explicit handoff, and outputs leave through an explicit collection step. Destroying the workspace prevents cross-task leakage and hidden state. Persist only product-needed artifacts, such as a validated chart or report.

## Where It Fits

Sandboxed execution is usually behind a controlled code-execution tool. The agent chooses whether to request the tool, but it does not choose the sandbox policy or receive the host's credentials.

```text
User request
     ↓
Agent proposes a code-execution action
     ↓
Policy and authorization check
     ↓
Sandbox manager creates one execution
     ↓
Isolated runtime runs code
     ↓
Validated result returns to the agent
```

For consequential work, the policy layer may require approval before it creates the sandbox. The durable workflow layer records the request and outcome. Observability records enough metadata to investigate a failure without unnecessarily retaining sensitive code or outputs.

## Tradeoffs

| Choice | What it improves | What it costs or risks |
| --- | --- | --- |
| Narrow application tool | Smallest authority and simplest audit | Cannot support open-ended code tasks |
| Native container with hardening | Fast startup and familiar operations | Shares the host kernel; weaker boundary for untrusted code |
| Application-kernel sandbox | Reduces direct host-kernel exposure | Some Linux behavior may be incompatible |
| MicroVM | Stronger workload isolation | Higher startup, memory, and operational cost |
| No network by default | Prevents many exfiltration and abuse paths | Jobs that need external data require explicit mediation |
| Fresh sandbox per job | Limits cross-task leakage and improves reproducibility | Image setup and cold-start latency |

Choose from the threat model, not latency alone. First decide the impact of an escape, wrong file read, or unexpected endpoint.

## Failure Modes

- **Sandbox in name only:** Generated code runs in the application worker, or a container has host mounts, a privileged mode, or a powerful socket attached.
- **Secret leakage:** A general cloud credential, environment variable, or metadata endpoint gives the code access far beyond its assigned task.
- **Open egress:** The job can upload inputs, scan internal services, download arbitrary payloads, or use external APIs without a policy decision.
- **Resource exhaustion:** Infinite loops, fork bombs, oversized output, or decompression bombs exhaust shared CPU, memory, disk, or log storage.
- **Cross-tenant residue:** A reused disk, cache, browser profile, or process exposes one task's data to another.
- **Weak result boundary:** The application blindly renders HTML, executes returned commands, or trusts an output file path supplied by the sandbox.
- **Assumed invulnerability:** A sandbox runtime vulnerability or configuration error permits an escape. Defense in depth and fast patching still matter.

## Example

A financial-analysis assistant lets analysts upload a spreadsheet and request a chart. The agent can ask for a `run_analysis` tool, but it cannot run shell commands on the service host.

The tool creates a new execution with the spreadsheet mounted read-only and an empty output directory. It allows Python and approved data libraries, caps memory and runtime, limits processes, and blocks the network. The task has no cloud credentials.

The script produces `chart.png` and a small JSON summary. The tool rejects other files, checks the JSON against a schema, and stores the validated artifacts. It destroys the sandbox afterward. If the script reads an absent path or contacts the internet, the operating environment denies it rather than trusting the model to obey instructions.

## Interview Takeaways

- An agent's ability to generate code is not an authorization decision. Run code behind a policy-controlled execution tool.
- A sandbox limits blast radius but does not replace input validation, authorization, approval, or secret management.
- Treat network access, filesystem mounts, credentials, CPU, memory, processes, and time as explicit capabilities.
- Use fresh workspaces, read-only inputs, constrained outputs, and cleanup to prevent cross-task leakage.
- Pick an isolation mechanism from the threat model. Containers, application-kernel sandboxes, and microVMs have different security and operational costs.

## Next

Next: [LLM Inference and Serving](../inference/llm-inference-and-serving.md). The same system must turn prompts into model output reliably, efficiently, and within capacity limits.

## Go Deeper

### WebAssembly as a Constrained Runtime

WebAssembly runtimes can suit small, controlled extensions: a module must explicitly receive the host functions, directories, and network interfaces it can use. This can simplify capability design. It is not a universal replacement for a VM; compatibility, native dependencies, and the runtime's security model still matter.

## References

- [OWASP: LLM06, Excessive Agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/)
- [NIST SP 800-190: Application Container Security Guide](https://csrc.nist.gov/pubs/sp/800/190/final)
- [gVisor: What is gVisor?](https://gvisor.dev/docs/)
- [Firecracker: Design and threat containment](https://github.com/firecracker-microvm/firecracker/blob/main/docs/design.md)
- [Wasmtime: Security](https://docs.wasmtime.dev/security.html)
