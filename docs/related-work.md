# Related Work

Pentimento's thesis is not that nothing exists - it is that everything that exists is either format-level, single-purpose, or dynamic, and nobody has unified the static weight-space methods into a practitioner toolchain.
This document is the evidence for that claim, and the map of what we build on.

## Serialization-level scanners (adjacent, not overlapping)

- **[modelscan](https://github.com/protectai/modelscan)** (Protect AI) - scans model files for unsafe code in serialization (pickle opcodes, H5/SavedModel payloads). Denylist-based; documented bypasses exist. Never inspects weight values semantically.
- **picklescan** - pickle-specific; has had its own bypass CVEs (see JFrog's zero-day writeups), which is an argument for defense in depth, not against scanning.
- **PickleBall** (arXiv:2508.15987) - secure deserialization via policy-constrained pickle loading; complements our metadata-only pickle frontend.

*Positioning*: these answer "will loading this file execute code?" Pentimento answers "what is in the weights once loaded?" Both belong in a deployment gate.

## Weight fingerprinting and lineage (our standard library)

- **SeedPrints** (arXiv:2509.26404, [code](https://github.com/YnezT0311/SeedPrints)) - fingerprints persisting from random initialization, tracing lineage even through heavy training.
- **AWM: Accurate Weight-Matrix Fingerprint** (arXiv:2510.06738, [code](https://github.com/LUMIA-Group/AWM)) - determines whether a suspect LLM derives from a base model via weight-matrix invariants.
- **GhostSpec / "Ghost in the Transformer"** (arXiv:2511.06390, [code](https://github.com/DX0369/GhostSpec)) - SVD-based invariant spectral signatures of attention products; robust to fine-tuning, pruning, expansion.
- **TensorGuard** (arXiv:2506.01631) - gradient-based family classification (note: requires backprop, so it sits at our static/dynamic boundary; included for completeness, not as a core backend).
- **Scalable fingerprinting** (arXiv:2502.07760, [code](https://github.com/SewoongLab/scalable-fingerprinting-of-llms)) - *active* fingerprint insertion by model owners; the complement of our passive detection (we verify, they mark).

*Positioning*: each is one method with one paper's repo.
Pentimento turns them into pluggable backends behind one CLI, one report format, and one calibration methodology - the way Ghidra absorbed decades of one-off RE techniques.

## Static backdoor detection (our rules/anomaly foundations)

- **Detecting Backdoored LoRAs from Weights Alone** (arXiv:2602.15195) - backdoors leave concentrated singular values with low spectral entropy in LoRA updates; five statistics per attention update suffice. Directly informs our rule feature set.
- **Static weight analysis for trojan detection** (Springer, ICANN'22 lineage; and the TrojAI program's final report, arXiv:2602.07152) - abnormal weight distributions on backdoor target labels; frozen-weights detection without training data.
- **Planting Undetectable Backdoors in ML Models** (Goldwasser, Kim, Vaikuntanathan, Zamir, FOCS 2022) - the impossibility boundary. This paper is why our [threat model](threat-model.md) has a Tier 3 and why Pentimento never claims completeness.

## Weight manipulation tools (constructive twins)

- **[mergekit](https://github.com/arcee-ai/mergekit)** (Arcee) - merging and LoRA *extraction* (`mergekit-extract-lora`). Extraction is the constructive twin of our low-rank graft *detection*; we share math, not purpose.
- **[ckpt](https://github.com/stef41/ckpt)** - checkpoint inspection/diff/validation without GPU loading. Closest existing utility in spirit; scope is per-tensor numeric diffing and file validation, without canonicalization, fingerprinting, signatures, or calibrated reporting.

## Symmetry, alignment, and numerics (foundations of the canon/WIR design)

- **Git Re-Basin** (Ainsworth et al., [arXiv:2209.04836](https://arxiv.org/abs/2209.04836)) - permutation-alignment via weight matching; validated on CNN/ResNet-scale models. We take the LAP machinery and the symmetry framing, and explicitly *do not* inherit its scope claim for transformers.
- **SliceGPT** (Ashkboos et al., ICLR 2024, [arXiv:2401.15024](https://arxiv.org/abs/2401.15024)) - the computational-invariance theorem: RMSNorm transformers are exactly invariant to per-block orthogonal rotation of the residual stream (LayerNorm convertible to RMSNorm form). This is the reason permutation-only canonicalization is insufficient and invariant features are our primary path.
- **Beyond the Permutation Symmetry of Transformers** ([arXiv:2502.00264](https://arxiv.org/abs/2502.00264)) and **Generalized Linear Mode Connectivity for Transformers** ([arXiv:2506.22712](https://arxiv.org/abs/2506.22712)) - rotation-aware alignment for transformers is current research; grounds our decision to tag alignment-mode findings with residuals.
- **CKA similarity** (Kornblith et al., [arXiv:1905.00414](https://arxiv.org/abs/1905.00414)) - orthogonal- and scale-invariant similarity; the metric inside AWM's pipeline.
- **Randomized SVD** (Halko, Martinsson, Tropp, [arXiv:0909.4061](https://arxiv.org/abs/0909.4061)) and **single-view sketching** (Tropp et al., [2017](https://users.cms.caltech.edu/~jtropp/reports/TYUC17-Randomized-Single-View-TR.pdf)) - streaming spectra with a priori error bounds, for tensors too large for exact per-tensor SVD.
- **Spectra of transformer weights under quantization/compression** ([Small Singular Values Matter, arXiv:2410.17770](https://arxiv.org/abs/2410.17770); [LQER, arXiv:2402.02446](https://arxiv.org/abs/2402.02446)) - LLM weight spectra are heavy-tailed and quantization distorts the tail; grounds the per-feature quantization validity envelopes.
- **[fickling](https://github.com/trailofbits/fickling)** (Trail of Bits) - static pickle decompilation/analysis; the existence proof for our metadata-only, never-execute pickle frontend.

## Interpretability (dynamic sibling)

Mechanistic interpretability (TransformerLens, SAE tooling, activation patching) answers deeper questions than any static method, but requires running the model, scales poorly to hub-wide screening, and is aimed at researchers.
Static screening and dynamic deep-dives are complements: Pentimento is the cheap always-on filter that tells you which checkpoints deserve expensive dynamic attention.

## The gap, stated precisely

No existing project combines: (1) semantic weight-value analysis, (2) inference-free operation, (3) more than one method under one interface, (4) community-extensible signatures, and (5) a published threat model with calibrated error rates.
Serialization scanners have (2) and (4) without (1); paper repos have (1) and (2) without (3), (4), (5); interp has (1) without (2).
That five-way intersection is Pentimento.
