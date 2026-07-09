# Threat Model

A static analysis tool for weights is a security tool, and a security tool without an explicit threat model is marketing.
This document states who Pentimento defends against, who it does not, and why the line is where it is.

## Assets and setting

The asset is an organization's (or the community's) trust in a checkpoint pulled from an untrusted or semi-trusted source: a public hub, a vendor, a fine-tuning contractor, an internal artifact store.
The question Pentimento answers is: **what can be learned about this checkpoint's contents, changes, and origin from the bytes alone, before anyone runs it?**

## Adversary tiers

### Tier 0 - the sloppy or dishonest uploader (in scope, primary)

Re-uploads a derivative of a licensed base model as "trained from scratch"; strips attribution; publishes a fine-tune of a fine-tune with fabricated model cards.
No active evasion effort.

*Coverage*: lineage fingerprinting and diff are designed to make this tier's claims cheaply checkable.
This is the overwhelming majority of real-world provenance problems today, and the tier where regulators and hub operators need tooling most.

### Tier 1 - the low-effort implanter (in scope, primary)

Applies a published or lightly modified attack: a known backdoor training recipe, a refusal-removal LoRA, a data-poisoning implant from a paper, a malicious layer transplant.
May rename tensors or requantize to obscure the change, but does not do weight-space engineering specifically to defeat static analysis.

*Coverage*: signature rules for known implant families, low-rank graft detection, and population-relative anomaly detection.
This is the YARA regime: signatures catch reuse of known techniques, which is how most attacks actually arrive.

### Tier 2 - the evasion-aware attacker (partially in scope)

Knows Pentimento exists and adapts: distributes an implant across many tensors to stay under per-tensor thresholds, matches the statistical profile of clean fine-tunes, launders lineage through merges, distillation, or heavy continued pretraining.

*Coverage*: partial and honest about it.
Invariant-based fingerprints raise the cost of lineage laundering (published methods survive fine-tuning, pruning, and quantization to documented degrees; we report those envelopes rather than pretending robustness is absolute).
Distillation, in particular, transfers behavior without transferring weights, and weight-space lineage claims do not extend to it - a distilled copy is out of reach of any weight-space method, by construction.
Anomaly detection forces this attacker to be statistically indistinguishable from the clean population, which constrains implant capacity.
It does not make attacks impossible; it makes them more expensive and more fragile.

### Tier 3 - the cryptographically careful adversary (explicitly out of scope)

Goldwasser, Kim, Vaikuntanathan and Zamir ("Planting Undetectable Backdoors in Machine Learning Models", FOCS 2022) show settings where a planted backdoor is computationally undetectable from weights, and weight-space steganography results show substantial payloads can be hidden below statistical noise floors.
**No static tool can promise detection against this tier, and any tool that does is lying.**
Pentimento's claim against Tier 3 is limited to: provenance verification (attestation of *where a checkpoint came from*) remains meaningful even when content inspection is defeated, which is why lineage tooling and content scanning ship together.

## Pentimento's own attack surface

A scanner that analysts point at malicious files is itself a target (this is the lesson of every antivirus and of picklescan's own CVEs):

- **No unpickling of untrusted input, ever.** PyTorch pickle checkpoints are parsed with a restricted opcode reader for metadata only; analyses requiring tensor values ask for the checkpoint in a safe format instead.
- **Parser hardening**: safetensors/GGUF headers are attacker-controlled input; loaders are fuzzed, and header claims are validated against file geometry before any offset is trusted (lesson imported from PE/ELF parsing history).
- **No network calls during analysis**; reference corpora and rule sets are explicit local inputs with content hashes.
- **Resource bounds**: adversarial headers must not be able to force pathological memory or compute (zip-bomb-equivalent defense).

## Failure modes we accept and publish

1. **False positives are a governed budget, not an embarrassment to hide** - every shipped detector carries a measured FP rate on held-out clean checkpoints, because an unbounded-FP scanner trains its users to ignore it.
2. **Novel attack families are invisible until a signature or statistical trace is characterized** - like YARA, the loop is: incident, analysis, rule, community distribution.
3. **Similarity evidence degrades under aggressive transformation** - each fingerprint backend documents the transformations it survives; outside that envelope, Pentimento reports "insufficient evidence", not a guess.

## Detection philosophy

Antivirus never promised to stop nation-states, and it still reshaped the economics of malware.
The goal here is identical: make the *cheap* attacks detectable, make the detectable attacks expensive, and give the ecosystem a shared language (rules, fingerprints, reports) for the arms race that follows.
Perfect static detection is provably impossible; useful static detection is provably absent today.
Pentimento targets the second problem.
