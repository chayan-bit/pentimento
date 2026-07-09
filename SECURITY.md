# Security Policy

Pentimento is a security tool, and security tools are targets: analysts will point it at attacker-controlled files.
We treat the following as security vulnerabilities in Pentimento itself:

- Any path by which analyzing a malicious checkpoint executes code (unpickle bypasses, parser memory corruption).
- Resource-exhaustion via adversarial headers (unbounded memory/compute from crafted safetensors/GGUF metadata).
- Rule-engine escapes (a rule file causing effects beyond feature evaluation).
- Report-layer injection (findings text from attacker-controlled tensor names breaking downstream consumers).

## Reporting

Report privately via GitHub Security Advisories ("Report a vulnerability" on the repo) rather than public issues.
Include the crafted file or a generator script if you can; we will credit you in the advisory unless you prefer otherwise.

## Out of scope

- Evasion of detection (a model that Pentimento fails to flag) is a detection-quality issue, not a vulnerability - file a regular issue, ideally with a rule proposal; see the arms-race discussion in `docs/threat-model.md`.
- Findings being interpreted as stronger claims than the report states - the report language is the contract.
