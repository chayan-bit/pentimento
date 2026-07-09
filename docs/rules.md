# Weight Signature Rules (draft spec)

The `pentimento.rules` engine is the YARA analog: detection knowledge as shareable, reviewable **data**, so a new implant family characterized by one analyst becomes a scan everyone runs the next day.
This is a draft; the fastest way to improve it is to try writing a rule for an attack you know and file an issue where the language fails you.

## Design requirements

1. **Architectural selectors, not filename selectors.** Attackers rename tensors for free; rules must select by *role* ("attention output projections, layers 20-30") resolved through the WIR's architecture graph, with raw-name patterns as a fallback only.
2. **Features, not raw bytes.** Weight bytes vary under precision/quantization; rules match on WIR features (spectral statistics, moments, delta-rank against a declared base) that have documented invariance properties.
3. **Declared error rates.** A rule does not enter the community set without a measured false-positive rate against the reference corpora (see [architecture.md](architecture.md), anomaly module).
4. **YARA-style metadata discipline**: severity, references, author, license, and the attack family the rule detects.

## Sketch

```yaml
rule: lora_refusal_removal_v1
meta:
  description: >
    Low-rank graft on attention q/v projections in mid-depth layers with
    spectral signature matching published refusal-removal fine-tunes.
  severity: high
  family: alignment-stripping/lora-graft
  references:
    - arxiv:2602.15195        # spectral statistics of backdoored LoRAs
  author: pentimento-community
  fp_rate: {corpus: llama-3.1-8b-ref-v1, measured: 0.4%}

requires:
  base: declared               # rule needs `pnt scan --base <checkpoint>`
  architecture: llama-family
  quantizations: [none, q8_0]  # feature validity envelope

select:
  role: attn.{q_proj,v_proj}
  layers: 40%..80%             # relative depth, architecture-independent

features:
  delta: canonical(self - base)      # after alignment pass
  rank_eff: effective_rank(delta, energy=0.95)
  sv_entropy: spectral_entropy(delta)
  sv_top_share: top_singular_energy(delta, k=4)

condition: >
  rank_eff <= 32
  and sv_entropy < corpus_percentile(sv_entropy, 1%)
  and sv_top_share > 0.85
  and count(matched_layers) >= 6
```

## Semantics notes

- `select` resolves through architecture templates; a rule that cannot resolve roles on a given model reports **not-applicable**, never silently passes.
- `requires.base` makes delta-features explicit: some rules only make sense as diffs against a claimed ancestor, and the report records which base was used.
- `corpus_percentile` ties conditions to empirical populations instead of magic constants, so thresholds are auditable and re-derivable.
- Every fired rule reports the *tensor coordinates* of the evidence, so findings are verifiable by hand.

## Non-goals

- Turing-complete conditions (rules must be statically analyzable and cheap to evaluate over streams).
- Behavioral claims ("this model will do X") - rules detect weight-space patterns; interpreting them is the analyst's job, and rule `meta.description` must be written as pattern description, not prophecy.
