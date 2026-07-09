# Architecture

Pentimento is organized like a classic binary-analysis toolchain: loaders feed a common intermediate representation, analyses run over the IR, and everything reports through one calibrated output layer.

```
 checkpoint files                 analyses                      output
┌──────────────────┐   ┌───────────────────────────┐   ┌──────────────────┐
│ safetensors      │   │ diff        (BinDiff)     │   │ report           │
│ GGUF             │──▶│ fingerprint (FLIRT)       │──▶│  - human text    │
│ PyTorch .bin*    │   │ scan        (YARA)        │   │  - JSON (SARIF-  │
│ ONNX (later)     │   │ anomaly     (entropy/     │   │    inspired)     │
└──────────────────┘   │              packer       │   │  - CI exit codes │
        │              │              heuristics)  │   └──────────────────┘
        ▼              └───────────────────────────┘
┌──────────────────┐               ▲
│ Weight IR (WIR)  │───────────────┘
│ + canonicalizer  │
└──────────────────┘
```

`*` PyTorch pickle files are loaded metadata-only through a restricted parser; Pentimento never unpickles untrusted files (see [threat-model.md](threat-model.md), "Pentimento's own attack surface").

## Modules

### `pentimento.loader`

Format frontends with one non-negotiable contract: **no analysis may materialize a full checkpoint in memory.**
The loader exposes tensors as memory-mapped, lazily-sliced views with a streaming iterator per layer.
This is what makes 70B+ models tractable on a laptop: safetensors and GGUF both store a header/index that lets us locate any tensor without reading the file, so per-tensor statistics stream at disk bandwidth.

Initial formats: safetensors, GGUF (including quantized blocks - analyses declare which quantizations they support).
Pickle-based PyTorch checkpoints get a metadata-only frontend built on a restricted opcode subset (fickling-style), because full unpickling of untrusted input is itself the classic model-file vulnerability.

### `pentimento.wir` - the Weight Intermediate Representation

The disassembly stage.
A loaded checkpoint becomes a **WeightGraph**:

- **nodes**: named tensors with shape, dtype, quantization, storage span, and streaming statistics (moments, kurtosis, entropy, extremal values, singular value sketches computed via randomized SVD on streamed blocks);
- **edges**: architectural relations (this tensor is the Q projection of attention block 12) recovered from naming conventions plus shape unification against a library of architecture templates (Llama, Mistral, Qwen, GPT-NeoX, ...);
- **metadata**: everything the file format claims about itself, kept separate from what we measured, so lies in metadata are themselves a finding.

Every analysis consumes the WIR, never raw files.
New formats only require a loader; new analyses only require the WIR.

### `pentimento.canon` - canonicalization

Weight space has symmetry groups under which functionally identical networks are numerically unrelated: neuron/head permutations, positive scaling across normalization boundaries, and rotations of the residual stream basis.
Any diff or similarity claim made without handling this is wrong, so canonicalization is a first-class pass, not a patch:

- **alignment mode**: find the permutation/scaling that best aligns two checkpoints (Git Re-Basin-style weight matching) before diffing;
- **invariant mode**: skip alignment entirely by computing features that are invariant to the symmetry group - singular value spectra, eigenvalue distributions, spectral signatures of attention products - used by fingerprinting.

Which mode applies, and which symmetries each feature is invariant to, is recorded in every report.

### `pentimento.diff`

Semantic diff of two WeightGraphs, assuming shared architecture (post-canonicalization):

- per-tensor delta norms and a touched-layer heat map ("this fine-tune concentrated on attention heads in layers 12-18");
- **low-rank structure recovery**: factor the delta and report effective rank, so a grafted LoRA is identified as "rank-16 update on q_proj/v_proj" rather than noise (mergekit's LoRA extraction is the constructive twin of this analysis);
- change classification heuristics: full fine-tune vs LoRA graft vs layer transplant vs quantization-only vs embedding surgery.

### `pentimento.fingerprint`

Lineage and provenance, built as pluggable backends implementing methods from the literature behind one interface: spectral/SVD signatures of attention matrices, weight-matrix invariants, and initialization-persistent fingerprints (see [related-work.md](related-work.md) for the specific papers).
Backends produce vectors; an index over reference checkpoints (public hubs) turns vectors into "nearest ancestors with confidence".
Reports state the method, its published robustness envelope (survives fine-tuning? pruning? quantization?), and calibrated confidence - never a bare percentage.

### `pentimento.rules`

The YARA analog: declarative signatures over WIR features, so detection knowledge is shareable data rather than code.
Rules select tensors (by architectural role, not just name), compute features (spectral entropy, singular value concentration, delta-rank against a declared base), and combine conditions with metadata (severity, references, license).
Full draft spec: [rules.md](rules.md).
The intended endgame is a community rules repository - signatures for known backdoor implant families, known refusal-removal patches, known merge fingerprints - with the same contribution dynamics as YARA rule sets.

### `pentimento.anomaly`

Population-relative outlier detection: per-architecture **reference corpora** are built by streaming feature extraction over public-hub checkpoints, giving empirical distributions for every WIR statistic.
A finding is only emitted with its measured false-positive rate on held-out clean checkpoints from the corpus.
No corpus for the architecture means the module reports "no baseline" - it never guesses from priors.

### `pentimento.report`

One output layer for everything: human-readable text, machine-readable JSON (SARIF-inspired: findings with rule IDs, locations as tensor coordinates, confidence, and provenance of the method), and CI-friendly exit codes so `pnt scan` can gate a deployment pipeline.
Calibration is a report-layer responsibility: every confidence number must trace to a documented calibration procedure.

## CLI

`pnt inspect | diff | lineage | scan | anomaly`, each a thin veneer over the Python API.
The CLI is the product for practitioners; the API is the product for platform integrations (hub-side scanning, CI gates).

## Design invariants

1. **Static only.** No analysis may execute the model or require a GPU. If a question needs inference, it is out of scope (and probably belongs in an interpretability tool).
2. **Streaming only.** O(largest tensor) memory, never O(model).
3. **Evidence, not verdicts.** Outputs are findings with methods and confidence, never legal or safety conclusions.
4. **Symmetry-aware by construction.** No raw-weight comparison ships without a documented invariance story.
5. **Measured error rates.** A detector without a published FP/FN evaluation on the reference corpora does not leave experimental status.
