# Roadmap

Phases are gated by exit criteria, not dates.
Everything before Phase 4 runs on a laptop against real hub checkpoints - no GPU anywhere.

## Phase 0 - design under fire (current)

Publish architecture, threat model, and rule-language draft; solicit adversarial review.

*Exit*: at least one external design review round absorbed; rule-language sketch validated against three known attack write-ups from the literature.

## Phase 1 - loader + WIR + `pnt inspect` / `pnt diff`

Streaming safetensors and GGUF frontends, the WeightGraph IR with streaming statistics, architecture template matching (Llama/Mistral/Qwen families first), alignment-based canonicalization, per-tensor diff with touched-layer maps and low-rank graft recovery.

*Exit*: correctly identifies rank and target modules of real LoRA fine-tunes from the hub (validated against their published adapter configs); 70B checkpoint inspected in O(largest tensor) memory; loader fuzzing in CI.

## Phase 2 - fingerprinting + `pnt lineage`

Pluggable fingerprint backends (spectral/SVD signatures, weight-matrix invariants, initialization-persistent prints - see [related-work.md](related-work.md)), a reference index built over a curated slice of public-hub checkpoints, calibrated nearest-ancestor reporting.

*Exit*: reproduce the headline lineage results of at least two published methods through our unified interface; measured precision/recall on a held-out set of hub models with *known* lineage; confidence calibration documented.

## Phase 3 - rules engine + `pnt scan` + community rules repo

Rule language v1 implementation, seed rule set for published implant families (starting with spectral LoRA-backdoor statistics from the literature), separate `pentimento-rules` repository with YARA-style contribution and review process, SARIF-ish JSON output and CI exit codes.

*Exit*: seed rules detect their target attacks reproduced from papers, with measured FP rates on clean corpora; one external rule contribution merged.

## Phase 4 - reference corpora + `pnt anomaly`

Hub-scale streaming feature extraction, per-architecture empirical distributions shipped as versioned corpus artifacts, population-relative outlier reporting with per-finding FP rates.

*Exit*: corpora for the top hub architectures; anomaly findings on a labeled evaluation set beat naive per-tensor thresholds at fixed FP budget.

## Phase 5 - ecosystem integrations

Hub-side scanning hooks, pre-deployment CI gate recipes, artifact-store plugins; optional signed attestation of scan results.
An on-chain commitment layer for weight attestation remains explicitly **out of the core** unless a concrete integration earns it - lineage evidence must stand on its own.

## Standing tracks (every phase)

- **Evaluation honesty**: every detector ships with FP/FN numbers or ships as `--experimental`.
- **Self-security**: parser fuzzing, no-unpickle policy, resource-bound enforcement (the scanner is a target - see [threat-model.md](threat-model.md)).
- **Paper watch**: new fingerprinting/backdoor-detection results get triaged as candidate backends or rule features.
