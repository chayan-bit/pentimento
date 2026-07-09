# Architecture

Pentimento is organized like a classic binary-analysis toolchain: loaders feed a common intermediate representation, analyses run over the IR, and everything reports through one calibrated output layer.

Every mechanism in this document carries a verification status: **[verified]** (published method or format spec, cited), **[derived]** (follows from a verified result by construction, derivation given), or **[open]** (our design, needs empirical validation before it leaves experimental status).
The summary table is at the [end of this document](#methodology-verification-status).

```
 checkpoint files                 analyses                      output
┌──────────────────┐   ┌───────────────────────────┐   ┌──────────────────┐
│ safetensors      │   │ diff        (BinDiff)     │   │ report           │
│ GGUF             │──▶│ fingerprint (FLIRT)       │──▶│  - human text    │
│ PyTorch .bin*    │   │ scan        (YARA)        │   │  - JSON (SARIF-  │
│ ONNX (later)     │   │ anomaly     (entropy/     │   │    inspired)     │
└──────────────────┘   │              packer       │   └──────────────────┘
        │              │              heuristics)  │
        ▼              └───────────────────────────┘
┌──────────────────┐               ▲
│ Weight IR (WIR)  │───────────────┘
│ + canonicalizer  │
└──────────────────┘
```

`*` PyTorch pickle files are loaded metadata-only through a restricted parser; Pentimento never unpickles untrusted files (see [threat-model.md](threat-model.md), "Pentimento's own attack surface").

## The symmetry problem, stated precisely

Everything downstream depends on getting this right, so it comes first.
A transformer's function is preserved under a large group of weight transformations; any two checkpoints related by an element of this group are the *same model* and must be treated as such.
The catalog, per symmetry:

1. **FFN hidden-unit permutation** - for an MLP block `W_down · φ(W_up · x)` (and the gated `W_gate` variant), any permutation `P` of the hidden dimension gives `(W_down P^T)(P W_up) = W_down W_up`-equivalent computation. Size: `d_ff!` per layer. **[verified]** - classic result, systematized by [Git Re-Basin (Ainsworth et al., 2022)](https://arxiv.org/abs/2209.04836).
2. **Attention-head permutation** - heads within a layer commute; permuting head blocks of `W_Q, W_K, W_V` with the matching block permutation of `W_O` is exact. **[verified]** - same family of results.
3. **Intra-head invertible transforms** - `softmax(x W_Q W_K^T x^T)` depends only on the product `W_Q W_K^T`, so `W_Q → W_Q M`, `W_K → W_K M^{-T}` for any invertible `M` (per head) is exact; identically `W_V W_O` for the value path. **[derived]** from the attention computation; this is the invariance [GhostSpec](https://arxiv.org/abs/2511.06390) builds its fingerprints on.
4. **Residual-stream orthogonal rotation** - in RMSNorm transformers, `RMSNorm(x Q) Q^T W = RMSNorm(x) W` for any orthogonal `Q`, so the *entire residual basis* can be rotated per block (embedding out, every block in/out, unembedding in) with exact functional equivalence; LayerNorm models convert to RMSNorm-equivalent form by absorbing scale/mean into adjacent matrices. **[verified]** - this is the "computational invariance" theorem that [SliceGPT (Ashkboos et al., ICLR 2024)](https://arxiv.org/abs/2401.15024) is built on, and quantization schemes like QuaRot exploit the same fact.
5. **Norm-scale absorption** - RMSNorm gain vectors can be pushed into adjacent projections, a diagonal continuous symmetry. **[verified]** - standard, used constructively by SliceGPT's LayerNorm→RMSNorm conversion.

Consequences we design around:

- **Raw parameter comparison across independently-published checkpoints is meaningless** - an adversary can apply (1)-(5) for free, and symmetry (4) alone is a `O(d_model)`-dimensional continuous group. Any tool diffing raw weights across untrusted provenance claims is broken by construction.
- **The original plan of "Git Re-Basin-style alignment" was insufficient and is hereby corrected**: Git Re-Basin's weight matching was validated on CNNs/ResNets, not LLM-scale transformers, and it only handles *permutations* - it cannot undo symmetry (4) at all. Follow-up work confirms permutation alone is too coarse for transformers and rotation must be modeled ([Beyond the Permutation Symmetry of Transformers, 2025](https://arxiv.org/abs/2502.00264); [Generalized Linear Mode Connectivity for Transformers, 2025](https://arxiv.org/abs/2506.22712)). **[verified correction]**
- Therefore: **invariant features are the primary path** (they quotient out the group analytically); **explicit alignment is a secondary, best-effort mode** with a documented failure envelope.

## Modules

### `pentimento.loader`

Format frontends with one non-negotiable contract: **no analysis may materialize a full checkpoint in memory** - the budget is O(largest tensor), enforced by the loader API returning lazily-sliced memory-mapped views only.

- **safetensors** - the format is an 8-byte little-endian header length, a JSON header mapping tensor names to dtype/shape/byte-offsets, then a flat data region; every tensor is addressable without reading the file, which is the property the whole streaming design rests on. Header claims (offsets, lengths, shape×dtype products) are validated against file geometry before any offset is dereferenced. **[verified]** - [format spec](https://github.com/huggingface/safetensors).
- **GGUF** - magic + version, a key-value metadata table (architecture, hyperparameters, tokenizer), then a tensor-info table (name, shape, ggml type, offset) and aligned data; block-quantized types (Q4_K, Q8_0, ...) are decoded block-streamwise by published layout. **[verified]** - [format spec](https://github.com/ggml-org/ggml/blob/master/docs/gguf.md).
- **PyTorch pickle** - static opcode-level analysis only (fickling-style decompilation of the pickle program), extracting tensor metadata and flagging non-storage opcodes; the pickle VM is never run. Analyses needing values ask for a safe-format copy. **[verified]** - [fickling (Trail of Bits)](https://github.com/trailofbits/fickling) demonstrates static pickle analysis is practical; picklescan's bypass CVEs are the argument for never executing.
- Loaders are fuzzed in CI; adversarial headers are bounded (allocation caps derived from file size) so a crafted index cannot force pathological memory. **[open]** - engineering commitment, validated by the fuzz corpus.

### `pentimento.wir` - the Weight Intermediate Representation

The disassembly stage: a loaded checkpoint becomes a **WeightGraph**.

**Nodes** are named tensors carrying shape, dtype, quantization type, storage span, and a feature block computed in one streaming pass per tensor:

- Moments/extremes/entropy: streamed exactly (Welford accumulators). **[verified]** - textbook.
- **Singular value spectra** - the workhorse feature. Two regimes:
  - *Exact*: a projection matrix at LLM scale is at most ~`d_model × d_ff` (e.g. 8192×28672 fp32 ≈ 0.9 GiB); that fits the O(largest tensor) budget, so per-tensor exact SVD (LAPACK `gesdd` on the mmapped view) is the default. **[derived]** - feasibility is arithmetic, no approximation needed.
  - *Sketched*: for tensors where exact SVD is too slow (embeddings, `vocab × d_model`), randomized range-finder SVD with power iterations, or true single-pass sketching when even one extra pass is unwanted; both have a priori error bounds, and the sketch parameters + bound are recorded in the WIR so downstream features know their precision. **[verified]** - [Halko-Martinsson-Tropp 2011](https://arxiv.org/abs/0909.4061); [Tropp et al., single-view low-rank approximation, 2017](https://users.cms.caltech.edu/~jtropp/reports/TYUC17-Randomized-Single-View-TR.pdf).
- **Quantization validity envelope** - spectral features on quantized weights are computed on block-dequantized streams, and are *not* assumed equal to the fp original: quantization approximately preserves the dominant singular structure while distorting the spectral tail (LLM weight spectra are heavy-tailed and slowly decaying, so tail statistics are the fragile ones - [random-matrix analyses of transformer spectra](https://arxiv.org/abs/2410.17770) and the quantization-error literature, e.g. [LQER](https://arxiv.org/abs/2402.02446), both make this concrete). Every feature therefore declares which quantizations it is measured to survive, and corpora include quantized/fp pairs to measure exactly that. **[derived + open]** - direction verified by the cited literature; the per-feature envelopes are ours to measure.

**Edges** are architectural roles (this tensor is `attn.q_proj` of block 12), recovered by template matching: config metadata (safetensors sidecar `config.json`, GGUF KV table) proposes an architecture family, then shape unification against the family template verifies it tensor-by-tensor; where names lie or are stripped, role recovery falls back to shape+position inference and reports the ambiguity instead of guessing silently. **[derived]** - metadata is present by spec; the adversarial-renaming fallback is measured against deliberately-mangled corpora. Metadata claims and measured facts are stored separately, so a lie in metadata is itself a finding.

### `pentimento.canon` - canonicalization

Two modes, honestly separated after the correction above:

- **Invariant mode (primary)** - compute features that are analytically constant on symmetry orbits, so no alignment is ever attempted:
  - singular values of any single matrix are invariant to symmetries (1), (2), (4), (5) (each acts by orthogonal/permutation factors on one side, and `σ(UAV) = σ(A)` for orthogonal `U, V`); **[derived]** - elementary linear algebra;
  - spectra of the *products* `W_Q W_K^T` and `W_V W_O` additionally quotient out the intra-head transforms (3), because `(W_Q M)(W_K M^{-T})^T = W_Q W_K^T`, and residual rotation (4) conjugates the product (`Q^T W_Q W_K^T Q`), preserving eigen/singular spectra; **[verified]** - this is precisely the GhostSpec construction;
  - CKA-based similarity of weight Grams, which is invariant to orthogonal transforms and isotropic scaling by construction. **[verified]** - CKA's invariance properties are established ([Kornblith et al., 2019](https://arxiv.org/abs/1905.00414)); its use on weights with LAP alignment is AWM's published pipeline.
- **Alignment mode (secondary, best-effort)** - when a rule or diff genuinely needs coordinates (e.g. localizing *which heads* changed between two checkpoints claimed to share provenance): permutations are recovered by solving a linear assignment problem per layer (Hungarian/Jonker-Volgenant on weight-correlation costs - the same LAP primitive AWM validates at LLM scale **[verified]**), and residual rotation by orthogonal Procrustes between matched projection stacks **[open]** - Procrustes is classical, but transformer-scale rotation recovery is current research ([2502.00264](https://arxiv.org/abs/2502.00264), [2506.22712](https://arxiv.org/abs/2506.22712)), so alignment-mode findings are always tagged with alignment residuals and downgraded to "insufficient evidence" when residuals are high.

Every report records which mode produced the evidence and which symmetries the evidence is invariant to.

### `pentimento.diff`

Semantic diff of two WeightGraphs.
The crucial scoping fact: **the dominant use case needs no canonicalization at all.**
A fine-tune diffed against its true base shares the base's coordinates (training does not apply symmetry transformations; SGD moves continuously from the shared init), so `Δ = W_ft - W_base` is directly meaningful.
Canonicalization matters only when comparing checkpoints of *disputed* provenance - and there, invariant-mode fingerprinting (below) is the right tool, with alignment-mode diff as a localizer afterward.
**[derived]** - and testable: Phase 1's exit criterion validates raw-delta diffing against thousands of hub fine-tunes with known bases.

Mechanics:

- per-tensor delta norms (relative Frobenius), streamed; touched-layer heat maps aggregated by architectural role;
- **low-rank structure recovery**: truncated SVD of `Δ` per target matrix; effective rank at 95% spectral energy, stable rank `‖Δ‖_F² / ‖Δ‖_2²`, and top-k energy concentration. A LoRA graft of rank r appears as an exactly-rank-r delta on its target projections and nothing elsewhere - the constructive inverse is mergekit's published `mergekit-extract-lora`, so recoverability is demonstrated in production, not conjectured. **[verified]** - [mergekit](https://github.com/arcee-ai/mergekit);
- change classification (full fine-tune vs LoRA graft vs layer transplant vs quantization-only vs embedding surgery) as decision rules over the delta-rank/locality/dtype profile. **[open]** - heuristics; ship with measured confusion matrices on labeled hub pairs.

### `pentimento.fingerprint`

Lineage and provenance, as pluggable backends that are *individually published methods*, unified behind one interface and one calibration methodology:

- **spectral product signatures** (GhostSpec-style): SVD of the invariant attention products per layer, concatenated into a compact signature; published robustness: fine-tuning, pruning, expansion. **[verified]** - [arXiv:2511.06390](https://arxiv.org/abs/2511.06390);
- **aligned-CKA similarity** (AWM-style): LAP head/unit alignment then unbiased CKA between weight matrices; published robustness: SFT, continued pretraining, RL, multimodal extension, pruning, upcycling; ~30 s per pair on consumer hardware. **[verified]** - [arXiv:2510.06738](https://arxiv.org/abs/2510.06738);
- **initialization-persistent traces** (SeedPrints-style): distributional traces of the random init that survive full training, giving birth-certificate lineage even across heavy training. **[verified]** as published - [arXiv:2509.26404](https://arxiv.org/abs/2509.26404); independent replication is a Phase 2 exit gate before we rely on it;
- the backend interface requires each method to declare: invariances (which of symmetries (1)-(5) it quotients), transformation envelope (what it is measured to survive), and a calibration dataset; the lineage index over reference checkpoints then reports nearest ancestors with method-attributed, calibrated confidence. **[open]** - the unification layer is ours; the Phase 2 exit criterion is reproducing at least two backends' headline results through it.

What fingerprinting cannot do, by construction: detect distillation (behavior transfer without weight transfer) - stated in the [threat model](threat-model.md), not buried here.

### `pentimento.rules`

The YARA analog: declarative signatures over WIR features (spec: [rules.md](rules.md)).
The feature vocabulary is exactly the WIR/diff/canon feature set above, so every rule inherits the invariance and quantization envelopes of the features it uses - a rule cannot accidentally claim more robustness than its weakest feature.
Seed rules implement published detectors, starting with backdoored-LoRA spectral statistics (concentrated singular values, low spectral entropy in adapter updates; five statistics per attention update suffice in the published evaluation). **[verified]** - [Detecting Backdoored LoRAs from Weights Alone](https://arxiv.org/abs/2602.15195); our generalization of those statistics to full fine-tune deltas is **[open]** and ships as experimental.

### `pentimento.anomaly`

Population-relative outlier detection: per-architecture reference corpora are built by the streaming feature extraction above over public-hub checkpoints, giving empirical distributions for every WIR statistic; findings are emitted only with their measured false-positive rate on held-out clean checkpoints, and thresholds are corpus percentiles, not constants.
**[open]** by design - the module's entire epistemic content *is* the measurement; it has no claims prior to its corpora, and it answers "no baseline" when a corpus is missing.

### `pentimento.report`

One output layer: human text, machine JSON (SARIF-inspired findings with rule IDs, tensor-coordinate locations, method provenance, invariance tags, and confidence), CI exit codes.
Confidence calibration is a report-layer responsibility: a number appears only if it traces to a documented calibration procedure on a named corpus.
Attacker-controlled strings (tensor names, metadata) are treated as untrusted in rendering (no injection into downstream consumers).

## CLI

`pnt inspect | diff | lineage | scan | anomaly`, each a thin veneer over the Python API.
The CLI is the product for practitioners; the API is the product for platform integrations (hub-side scanning, CI gates).

## Design invariants

1. **Static only.** No analysis may execute the model or require a GPU. If a question needs inference, it is out of scope.
2. **Streaming only.** O(largest tensor) memory, never O(model); enforced at the loader API boundary.
3. **Evidence, not verdicts.** Outputs are findings with methods and confidence, never legal or safety conclusions.
4. **Symmetry-aware by construction.** No comparison ships without a documented position in the symmetry catalog above; invariant mode is preferred to alignment mode wherever possible.
5. **Measured error rates.** A detector without a published FP/FN evaluation on the reference corpora does not leave experimental status.

## Methodology verification status

| Mechanism | Status | Basis |
|---|---|---|
| safetensors/GGUF random tensor access from header | verified | format specs |
| No-unpickle static pickle metadata parsing | verified | fickling (Trail of Bits) |
| Transformer symmetry catalog (permutation/head/intra-head/rotation/scale) | verified | Git Re-Basin; SliceGPT computational invariance; attention algebra |
| Permutation-only alignment suffices for transformers | **refuted - corrected** | 2502.00264, 2506.22712; SliceGPT rotation symmetry |
| Exact per-tensor SVD within O(largest tensor) | derived | arithmetic on LLM tensor sizes |
| Sketched/single-pass SVD with a priori error bounds | verified | Halko-Martinsson-Tropp 2011; Tropp et al. 2017 |
| Singular spectra invariant to catalog symmetries | derived | σ(UAV)=σ(A); product invariances |
| Spectral product fingerprints robust to fine-tune/prune | verified (published) | GhostSpec 2511.06390 |
| LAP + CKA lineage similarity at LLM scale | verified (published) | AWM 2510.06738 |
| Init-persistent lineage traces | verified (published), replication gated | SeedPrints 2509.26404 |
| Rotation recovery (Procrustes) at transformer scale | open | active research area |
| Raw-delta diff valid for true base/fine-tune pairs | derived, Phase 1 exit test | shared-init continuity; mergekit LoRA extraction as constructive proof |
| LoRA-backdoor spectral rule features | verified (published) | 2602.15195 |
| Generalizing those features beyond LoRA | open | ships experimental |
| Quantization preserves leading spectrum, distorts tail | derived from literature | 2410.17770, LQER; envelopes measured per feature |
| Change-type classification heuristics | open | confusion matrices required |
| Anomaly FP-rate governance | open by design | is itself the measurement |
