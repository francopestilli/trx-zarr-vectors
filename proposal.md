## Proposal

Add a small storage-backend seam to trx-python so a `TrxFile` can also open and save a [Zarr Vectors](https://github.com/AllenInstitute/zarr-vectors-py) streamline store (`.zarrvectors`). The TRX spec does not change, the public `TrxFile` API does not change, and the Zarr dependency stays behind an optional `trx-python[zarr]` extra.

## What TRX users gain

Spatial queries (`TrxFile.query(bbox)`), range-request reads from S3 or GCS without downloading the file, and multi-resolution pyramids for visualisation, all through the `TrxFile` they already use. Every TRX-aware tool (DIPY, scilpy, bridges to other software) reads `.zarrvectors` streamline stores for free.

The two data models already line up one-to-one: `positions` + `offsets` ↔ `vertices` + `object_index`, `dpv` ↔ `vertex_attributes`, `dps` ↔ `object_attributes`, named `groups` ↔ `groups` with `group_names`, `dpg` ↔ `group_attributes`, and `header.json` ↔ `headers/trx/`. Zarr Vectors' read result (`positions` + `parts` slices) is the same shape as the nibabel `ArraySequence` that `TrxFile` holds, so reads need no copy.

## What would change in trx-python

| Change | Size | Behaviour change |
| --- | --- | --- |
| `trx/backends/` with a `TrxBackend` protocol; today's zip and folder code moves behind `MemmapBackend` | Medium | None; existing tests are the acceptance test |
| `ZarrVectorsBackend`: open, header, `read(indices)`, `query(bbox)`, groups, dpg | Medium | Additive, behind the `[zarr]` extra |
| `load()` detects a Zarr v3 group; `save()` writes `.zarrvectors` on extension or `backend=` | Small | Additive |
| `trx convert` accepts `.zarrvectors` as source and target | Small | Additive |
| Round-trip tests: every TRX dtype, sparse dpg, empty file, named groups, byte-identical | Medium | None |

The memmap path stays exactly as fast and as lazy as it is. On the Zarr backend a `TrxFile` holds no memmap; arrays are materialised per `read` or `query`, and in-place `resize` or `append` raise with a pointer to `write`.

## The one decision I need from you

In-tree `trx/backends/` package, or a separate `trx-zarr` package that registers a backend? In-tree keeps one test suite and one release; separate keeps trx-python's dependency list untouched. The design below assumes in-tree but works either way.

## How I would land it

1. This issue, for the decision above.
2. PR #152: the `MemmapBackend` refactor alone, no behaviour change (already open, so the shape of the seam can be judged on real code).
3. PR: `ZarrVectorsBackend` read path with fixture files.
4. PR: write path and `trx convert` target.

Each PR is reviewable in one sitting, and the first merges before anything Zarr-related exists. The Allen Institute side (dtype guarantees, a stable API list, a streaming writer) is being raised in parallel on zarr-vectors-py.

## Full design

The complete design (data-model mapping, where the two models disagree and the proposed conventions, backend architecture, per-project changes, phased plan, open questions) is at https://github.com/francopestilli/trx-zarr-vectors/blob/main/DESIGN.md. Comments on it are welcome here or as issues on that repository.
