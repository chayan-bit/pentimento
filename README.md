# Pentimento

**Static analysis toolchain for neural network weights - inspect, diff, fingerprint, and scan model checkpoints without running them.**

> *Pentimento* (art history): a visible trace of an earlier painting beneath the layers of a finished work.
> Fine-tunes, merges, grafts, and implants all leave pentimenti in weight space.
> This project builds the tooling to see them.

**Status: design phase.**
The architecture, threat model, and rule language are specified in [`docs/`](docs/); implementation is tracked in the [roadmap](docs/roadmap.md).
Design review issues and PRs are welcome now - this is exactly the stage where community scrutiny is cheapest.

---

## The problem

We treat model weights the way the industry treated binaries in 1995: as opaque blobs we either run or don't.

Every security and provenance question about a checkpoint today is answered *dynamically* - run it, probe it, eval it.
But dynamic analysis of a model is insufficient for the same reason dynamic analysis of a binary is: a backdoor that triggers on one rare input will never appear in your eval suite, exactly like malware that sleeps until a date check passes.
Meanwhile the supply chain has exploded: millions of unvetted checkpoints on public hubs, fine-tunes of fine-tunes with no recorded lineage, and enterprises deploying downloaded weights with no equivalent of an antivirus scan.

What binary security built over 30 years - disassemblers, differs, signature scanners, provenance databases - simply does not exist for weights:

- Existing model scanners ([modelscan](https://github.com/protectai/modelscan), picklescan) check the **serialization format** for code-execution payloads. They never look at the weights semantically.
- Mechanistic interpretability looks inside models, but requires running them, and is aimed at science rather than practitioner tooling.
- Model-merging tools ([mergekit](https://github.com/arcee-ai/mergekit)) manipulate weights but do not analyze them.
- Academic work on weight fingerprinting and static backdoor detection exists, but as scattered one-off paper repos, not a unified, maintained toolchain.

Nobody has built the IDA Pro. Pentimento is that missing static toolchain.

## What Pentimento is

Five capabilities over a shared intermediate representation, all operating on checkpoint files directly - no GPU, no inference, no training data required.

| Capability | Binary-security analog | What it answers |
|---|---|---|
| **Inspect** | disassembler / `readelf` | What is actually in this checkpoint? Architecture, tensor inventory, dtype/layout anomalies, metadata inconsistencies. |
| **Diff** | BinDiff | What did this fine-tune actually change? Per-tensor deltas, touched-layer maps, low-rank structure recovery ("this is a rank-16 LoRA grafted onto layers 12-18"). |
| **Fingerprint** | FLIRT signatures / ssdeep | Where did this model come from? Invariant lineage fingerprints that survive fine-tuning, permutation, and precision changes. |
| **Scan** | YARA | Does this checkpoint match a known-bad or known-notable pattern? Declarative community rules over weight-space features. |
| **Anomaly** | packed-section / entropy heuristics | Does anything here deviate from the empirical population of checkpoints with this architecture? |

Full module design: [docs/architecture.md](docs/architecture.md).
Rule language: [docs/rules.md](docs/rules.md).

## What Pentimento is not (read this before filing the obvious issue)

We have written down the limits up front, because a security tool that overclaims is worse than no tool.

1. **It is not a backdoor oracle.**
   Goldwasser, Kim, Vaikuntanathan and Zamir (2022) proved that cryptographically *undetectable* backdoors can be planted in some settings.
   No static tool can promise to catch a maximally resourced adversary, and we do not.
   Like YARA and antivirus, the goal is to **raise attacker cost** and reliably catch *known implant families and low-effort attacks* - which is the overwhelming majority of real-world abuse.
   See the [threat model](docs/threat-model.md) for the explicit adversary tiers we do and do not cover.

2. **It does not output legal conclusions.**
   Weight similarity is *evidence*, not proof.
   Pentimento reports calibrated findings ("spectral fingerprint match to Llama-3.1-8B, confidence 0.97, method AWM-style invariants") and never strings like "license violation."
   Turning evidence into a claim is a job for humans with lawyers.

3. **Naive diffing is known to be broken, so we don't do it.**
   Weight space has large symmetry groups - neuron/head permutations, scaling, and full orthogonal rotations of the residual stream (the computational-invariance result underlying SliceGPT) - under which two functionally identical checkpoints look numerically unrelated.
   Permutation-only alignment (Git Re-Basin-style) is provably insufficient for transformers, so Pentimento's primary path is **invariant features** (singular value spectra, invariant attention-product signatures, CKA) that quotient out the symmetry group analytically, with explicit alignment as a residual-tagged best-effort mode.
   The full symmetry catalog, with per-claim verification status, is in [docs/architecture.md](docs/architecture.md).

4. **"Statistical outlier" requires a reference population, not vibes.**
   Anomaly detection is defined strictly relative to empirical reference corpora built from public-hub checkpoints of the same architecture, shipped with measured false-positive rates.
   No corpus for your architecture yet means Pentimento says "no baseline" instead of guessing.

5. **It must work at 70B+ scale, so nothing ever loads a full model.**
   Every analysis is defined over streaming, memory-mapped, layer-at-a-time access to safetensors/GGUF - the loader contract forbids materializing the checkpoint.

## How it compares

| | Serialization scanners (modelscan, picklescan) | Merge/manipulation tools (mergekit, ckpt) | Academic fingerprinting repos (SeedPrints, AWM, GhostSpec) | Mech interp | **Pentimento** |
|---|---|---|---|---|---|
| Looks at weight *values* semantically | ✗ | partial | ✓ | ✓ | ✓ |
| Runs without inference | ✓ | ✓ | mostly | ✗ | ✓ |
| Unified toolchain, not one paper's method | ✓ | ✓ | ✗ | ✗ | ✓ |
| Community-extensible signatures | ✗ | ✗ | ✗ | ✗ | ✓ |
| Honest threat model published | partial | n/a | rarely | n/a | ✓ |

Pentimento treats the academic work as its standard library, not competition: fingerprinting methods from the literature become pluggable backends behind one CLI and one report format.
Citations and per-paper positioning: [docs/related-work.md](docs/related-work.md).

## Planned interface

```console
$ pnt inspect model.safetensors
$ pnt diff base.safetensors finetune.safetensors        # touched layers, low-rank grafts
$ pnt lineage mystery-model/ --index hub-reference.idx  # nearest ancestors + confidence
$ pnt scan model.gguf --rules community-rules/          # YARA-style weight signatures
$ pnt anomaly model.safetensors --corpus llama-8b.ref   # population outliers + FP rate
```

Everything also exists as a Python API, and reports emit machine-readable JSON (SARIF-inspired) for CI gates.

## Documentation

| Doc | Contents |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Module design, the Weight IR, canonicalization, streaming loader contract |
| [docs/threat-model.md](docs/threat-model.md) | Adversary tiers, in/out of scope, theoretical limits, detection philosophy |
| [docs/rules.md](docs/rules.md) | The weight-signature rule language (draft spec) |
| [docs/related-work.md](docs/related-work.md) | Annotated prior art: papers, tools, and what Pentimento borrows from each |
| [docs/roadmap.md](docs/roadmap.md) | Phased plan from docs-stage to hub-scale scanning |
| [docs/faq.md](docs/faq.md) | Preemptive answers to the sharpest objections |

## Contributing

Design-stage contributions wanted: adversarial review of the threat model, rule-language feedback, fingerprinting methods we missed, and reference-corpus methodology.
See [CONTRIBUTING.md](CONTRIBUTING.md) and [SECURITY.md](SECURITY.md).

## License

[Apache-2.0](LICENSE).
