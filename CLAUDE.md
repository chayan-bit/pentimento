# CLAUDE.md - Pentimento

Static analysis toolchain for neural network weights: inspect, diff, fingerprint, scan, and anomaly-check checkpoints without running them.
Currently in **Phase 0 (design)** - see `docs/roadmap.md`; the repo is docs-first until Phase 1 starts.

## Read before changing anything

- `docs/architecture.md` - module layout, the Weight IR, and the five design invariants (static-only, streaming-only, evidence-not-verdicts, symmetry-aware, measured error rates).
- `docs/threat-model.md` - adversary tiers; any capability claim you write must be consistent with it.
- `docs/related-work.md` - do not reinvent a method that exists as a paper; add it as a backend and cite it.

## Non-negotiable project rules

1. **Never overclaim.** No output, doc, or comment may promise complete backdoor detection or emit legal conclusions ("license violation"). Findings are evidence with method + calibrated confidence. This is the project's core credibility asset.
2. **Static only.** No analysis may run inference or require a GPU. If a proposed feature needs a forward pass, it is out of scope.
3. **Streaming only.** O(largest tensor) memory. Never materialize a full checkpoint; loaders expose mmap/streamed views.
4. **Never unpickle untrusted input.** Pickle checkpoints get the restricted metadata-only parser. No exceptions, including tests and quick scripts.
5. **Symmetry-aware.** Any new diff/similarity feature must document its invariance story (aligned, invariant, or explicitly neither).
6. **Measured error rates.** New detectors ship behind `--experimental` until FP/FN rates are measured against reference corpora.

## Code conventions (Phase 1+)

- Python 3.11+, typed; core numerics may drop to Rust later - keep the loader/WIR boundary clean for that.
- Immutability: analyses take WIR views, return new result objects; never mutate loaded state.
- TDD, >=80% coverage; loader code additionally gets fuzz targets (adversarial headers are the main attack surface - see threat model, "Pentimento's own attack surface").
- Files 200-400 lines (800 max), functions <50 lines; organize by module (`loader/`, `wir/`, `canon/`, `diff/`, `fingerprint/`, `rules/`, `anomaly/`, `report/`).
- Conventional commits (`feat:`, `fix:`, `docs:`...), no Co-Authored-By lines.

## Docs conventions

- Long Markdown: one full sentence per line.
- Plain dash `-`, never em dash.
- Every quantitative or capability claim in docs needs either a citation (add it to `docs/related-work.md`) or a pointer to the evaluation that will validate it.
- The FAQ (`docs/faq.md`) is the immune system: when a new objection appears in an issue, the fix usually includes an FAQ entry.
