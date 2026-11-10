# Status

Last updated: 2026-11-10

Tracks the work to let trx-python use Zarr Vectors as a storage backend. The design is in [DESIGN.md](DESIGN.md); the issue body sent to the TRX maintainers is [proposal.md](proposal.md).

## Done

- Design written: TRX spec unchanged, trx-python gains a pluggable storage backend, Zarr Vectors becomes a second container. Six conventions to agree (vertex order, dtype fidelity, booleans, sparse dpg, affine semantics, empty files) and a five-phase gated plan.
- zarr-vectors-py: PR from `francopestilli/zarr-vectors-py` branch `docs/trx-comparison-accuracy` rewriting `docs/spec/comparisons/trx_format.md` to match the TRX spec (raw binary array files, `dpv/` `dps/` `dpg/`, `groups/<name>.uint32`, the four header fields, positions already RASmm) and fixing the TRX links and a dps/dpp slip on the streamline page.
- zarr-vectors-py: issue opened on dtype fidelity (float16 vertices and integer attributes under `validate()`, bool convention, `present_mask` on `group_attributes`).
- trx-python: issue [#153](https://github.com/tee-ar-ex/trx-python/issues/153), the proposal, with the one decision for the maintainers: in-tree `trx/backends/` or a separate package.
- trx-python: PR [#152](https://github.com/tee-ar-ex/trx-python/pull/152), the `MemmapBackend` refactor. New `trx/backends/` package with a `TrxBackend` protocol and registry; zip/folder loading and saving moved verbatim into `MemmapBackend`; every public name in `trx.trx_file_memmap` kept. Existing suite 205 passed before and after, plus 6 new tests.
- Local clones: `~/git/zarr-vectors-py` (fork, `upstream` set), `~/git/trx-python` (fork updated from 2021 to current `main`, renamed from `tractography_file_format`, `upstream` set), `~/git/trx-zarr-vectors` (this repo).

## Open, on us

| Item | Where | Action |
| --- | --- | --- |
| PR comment on #152 says `#NNN` | trx-python | `gh pr comment 152 --edit-last` with `#153` |
| Dtype-fidelity issue links the Claude doc | zarr-vectors-py | Edit first sentence to `(proposal: tee-ar-ex/trx-python#153)` |
| Stability and streaming-writer issues | zarr-vectors-py | Open them (texts drafted) |
| Comparison-page PR | zarr-vectors-py | Confirm it opened after `git fetch upstream` |
| CI on #152 | trx-python | Watch ruff, codespell and the test matrix |

## Open, waiting on maintainers

- tee-ar-ex: in-tree vs separate package (#153); review of #152.
- Allen Institute: do float16 vertices and integer attributes validate; bool and `present_mask` conventions; which `api` names are frozen through 1.0; is a streaming polyline writer in scope.

## Next code

- Phase 2: `ZarrVectorsBackend` read path (open, header from `headers/trx/`, `read(indices)`, `query(bbox)`, groups and dpg) behind a `trx-python[zarr]` extra, with TRX fixtures covering every dtype. Location depends on the #153 answer.
- Phase 3: write path and `trx convert` target. Needs the streaming writer, or a decision to build on the `building` API.
- Phase 4: `zarr-vectors-tools` converters wrap the backend; lossy dtype handling retired.

Phases 2-4 are blocked on the two maintainer replies, not on code.
