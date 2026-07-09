# Contributing

Pentimento is in the design phase, which is the highest-leverage moment to contribute: changing a spec costs a paragraph, changing shipped behavior costs a migration.

## What we want right now (Phase 0)

- **Adversarial review of the threat model** (`docs/threat-model.md`): an evasion path we misfiled in the wrong tier, an adversary we didn't name, an overclaim that survived editing.
- **Rule-language stress tests** (`docs/rules.md`): take a published attack (see `docs/related-work.md` for starting points) and try to express a detector for it; file an issue wherever the language fails you.
- **Prior art we missed**: papers or tools that belong in `docs/related-work.md`, especially fingerprinting methods that could become backends.
- **Reference-corpus methodology**: how to sample public-hub checkpoints per architecture so FP-rate measurements mean something.

## Ground rules

- **Claims need evidence.** Doc changes that add capability claims must cite a source or a planned evaluation. "It should be possible to..." gets a design issue, not a merged sentence.
- **The tone is calibrated.** PRs that add marketing language ("guarantees", "detects all", "proves") will be asked to weaken it. This is by design; see the FAQ.
- **One topic per PR**, conventional commit messages (`docs:`, `feat:`, `fix:`), no Co-Authored-By trailers.
- Long Markdown uses one full sentence per line (it makes diffs reviewable); plain dash `-`, no em dash.

## When code lands (Phase 1+)

TDD with >=80% coverage; loaders additionally require fuzz targets; every design invariant in `docs/architecture.md` is enforced in review.
Security-sensitive reports (parser bugs, resource exhaustion, unpickle bypasses) go through [SECURITY.md](SECURITY.md), not public issues.

## License

Contributions are accepted under [Apache-2.0](LICENSE).
