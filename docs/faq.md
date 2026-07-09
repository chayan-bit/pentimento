# FAQ - the sharpest objections, answered first

### "Static backdoor detection is provably impossible. Isn't this project dead on arrival?"

Undetectable backdoors exist (Goldwasser et al., FOCS 2022) - against a cryptographically careful adversary, yes, and our [threat model](threat-model.md) says so in bold.
But "perfect detection is impossible" has never been an argument against building detection: the same theorem-shaped objection applies to antivirus, IDS, and YARA, all of which reshaped attacker economics anyway.
Real-world attacks overwhelmingly reuse published techniques, and published techniques leave characterizable traces.
The dead-on-arrival position has to explain why the *binary* world's static tooling was worth building; whatever answers that, answers this.

### "Two functionally identical networks can have wildly different weights. Diffing weights is meaningless."

Correct for *naive* diffing, which is why Pentimento doesn't ship one.
Diff runs after an alignment pass over the known symmetry groups (permutation/scaling/rotation), and fingerprinting uses features invariant to those groups (spectral methods) rather than raw parameters.
Every report states which invariance its evidence carries.
See [architecture.md](architecture.md), canonicalization.

### "94% weight overlap doesn't prove a license violation."

Agreed unreservedly - and Pentimento will never emit the words "license violation."
It emits evidence: matched fingerprint, method, robustness envelope, calibrated confidence, tensor coordinates.
Humans and lawyers turn evidence into claims.
The original pitch for this project used that phrasing; treating it as an output was the first flaw we removed.

### "A distilled model has none of the original weights. Your lineage tooling misses it."

Yes, by construction - distillation transfers behavior, not parameters, and *no* weight-space method can catch it.
We say so in the threat model rather than hand-waving.
Behavioral fingerprinting is a fine dynamic complement; it is out of scope here precisely because our value proposition is inference-free operation.

### "Won't attackers just test their implants against Pentimento until they pass?"

Some will - exactly as malware authors test against VirusTotal, and yet signatures remain the workhorse of practical detection.
Evasion pressure is the sign the tool is priced into attacks; the answers are the same as in binary security: population-relative anomaly detection (you must blend into the *empirical* distribution, not just pass point checks), private rule sets alongside public ones, and a community loop that turns each discovered evasion into a new rule.

### "Why not just run evals on the model?"

Run them - and note what they can't see.
Evals sample the input space; a trigger fired by a rare token sequence has essentially zero probability of appearing in your suite (the sleeping-malware problem).
Evals also cost GPU time per model, which is untenable for hub-scale screening of millions of checkpoints.
Static analysis is the cheap first-pass filter that decides which models deserve expensive dynamic scrutiny - the same division of labor as static/dynamic analysis of binaries.

### "The anomaly detector will drown people in false positives."

An unbounded-FP scanner trains users to ignore it, which is why FP rates are a governed budget in this project: no detector leaves experimental status without measured rates on held-out clean corpora, findings carry their rates, and thresholds are defined as corpus percentiles that anyone can re-derive.
See design invariant 5 in [architecture.md](architecture.md).

### "Isn't this dual-use? A lineage detector teaches evaders what to launder."

The information asymmetry today favors attackers *completely*: hiding a derivative currently requires no effort at all, because nobody checks.
Publishing detection methods with honest robustness envelopes raises the floor for everyone - the standard disclosure logic of the security field, and the same trade YARA, Sigma, and every AV vendor already made.
We additionally keep the door open for private rule sets where publication would burn a detection edge.

### "Why would this succeed as a unified toolchain when the methods are already public?"

Public methods without a toolchain don't get used - that is the actual lesson of Ghidra and IDA: the techniques existed in papers for decades; the products made them practice.
Five one-off paper repos with different interfaces, no calibration, no maintenance, and no shared report format are not infrastructure.
One CLI, one IR, one rule language, and one evaluation methodology are.

### "Where does the Web3/on-chain part fit?"

It doesn't, unless it earns its place - the roadmap keeps on-chain weight commitments explicitly out of the core.
Signed scan attestations and content-hash provenance work fine without a chain; if a concrete integration ever adds real verification value, it will arrive as an optional layer, not a dependency.
