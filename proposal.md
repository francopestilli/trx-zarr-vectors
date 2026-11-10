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
2. PR: the `MemmapBackend` refactor alone, no behaviour change.
3. PR: `ZarrVectorsBackend` read path with fixture files.
4. PR: write path and `trx convert` target.

Each PR is reviewable in one sitting, and the first merges before anything Zarr-related exists. The Allen Institute side (dtype guarantees, a stable API list, a streaming writer) is being raised in parallel on zarr-vectors-py.

<details>
<summary>Full design document</summary>


## Summary

Give trx-python a pluggable storage layer so a `TrxFile` can open and save a Zarr Vectors streamline store (`.zarrvectors`) exactly as it opens a `.trx` zip or folder today. The TRX specification does not change: TRX remains the community data model (positions, offsets, dpv, dps, groups, dpg, header), and Zarr Vectors becomes a second container for that model, alongside the existing memmap container.

The two models already line up component for component, and the Allen converters (`zarr-vectors-tools`) prove the round trip works. What is missing is a first-class backend inside trx-python so that users get lazy, spatially indexed, cloud-readable access through the API they already use, with no dtype loss and no full in-memory copy. This document proposes that backend, lists the small asks of each project, and lays out a phased plan.

All code references are to the repositories as of 2026-11-10: trx-python at the 2026-10-16 commit on `main`, zarr-vectors-py at format version 0.9.2 (2026-11-05).

## Why a backend, not a merger

TRX and Zarr Vectors optimise for different access patterns, and users increasingly need both on the same tractogram. TRX is a flat, memory-mapped layout: sequential and whole-file access is as fast as the disk, the spec is small, and it is already read by DIPY, scilpy and others. Zarr Vectors is spatially chunked on Zarr v3: a bounding-box query touches only the chunks it needs, chunks can be fetched by HTTP range request from S3 or GCS, and a store can carry coarser pyramid levels for visualisation.

A converter alone does not solve this. Today a user who wants spatial or cloud access has to run `ingest_trx`, then switch to the `zarr_vectors` API, and run `export_trx` to get back to the tools they already use. Every hop is a full copy, and the current converters widen or narrow dtypes on the way (see *Where the models disagree*). A backend inside trx-python means one API, one object, and the container chosen by the file you open.

A spec merger is premature. Zarr Vectors is at format version 0.9.2 with a hard break at 0.9.0 (per-chunk arrays became cells of a single vlen-bytes array; `cross_chunk_links/` merged into `links/`). TRX has been stable since 2022 and is a community-approved exchange format. Binding the TRX spec to a moving layout would import that churn. A backend keeps the dependency one way: trx-python depends on `zarr-vectors-py`'s public `api` surface, and nothing in TRX depends on the Zarr Vectors spec.

The backend also gives Zarr Vectors something: every TRX-aware tool gains a reader for `.zarrvectors` streamline stores for free, which is an adoption path into the diffusion MRI ecosystem.

## Data-model mapping

Every TRX component has a home in a Zarr Vectors `streamline` store; nothing is dropped and only two mappings need a convention.

| TRX component (spec) | Shape / dtype in TRX | Zarr Vectors home (level 0) | Fidelity |
| --- | --- | --- | --- |
| `header.json` VOXEL_TO_RASMM, DIMENSIONS | 4x4 float, 3 uint16 | `headers/trx/` attributes (`HeaderRegistry.add("trx", …)`) | Exact, verbatim JSON |
| `header.json` NB_STREAMLINES, NB_VERTICES | uint32, uint64 | Derived: `object_index` slot count and level `vertex_count` | Exact |
| `positions.3.<float>` | (V, 3) float16/32/64, RASmm | `vertices/` cells, in chunk order | Exact values; order differs (see next section) |
| `offsets.<uint>` | (S+1,) uint32/64 with sentinel | `object_index/manifests` row per streamline, ordered (chunk, fragment) refs | Exact; recomputed on read |
| `dpv/<name>.<N>.<dtype>` | (V, N) any accepted dtype | `vertex_attributes/<name>/` rows aligned 1:1 with `vertices/` | Exact if the store keeps the dtype |
| `dps/<name>.<N>.<dtype>` | (S, N) any accepted dtype | `object_attributes/<name>/` (O,) or (O, C) | Exact |
| `groups/<name>.uint32` | (K,) streamline indices | `groups/` ragged row g, with `group_names` on the array metadata | Exact; names are supported since the `GroupCatalog` API |
| `dpg/<group>/<name>.<N>.<dtype>` | (1, N) per group, keys optional per group | `group_attributes/<name>/` (G,) or (G, C), fill value for absent keys | Needs a convention for missing keys |
| dtype `bit` (boolean) | packed/bool file | uint8 or bool array | Needs a convention |

Two facts make this cheap to build on. First, `read_polylines` and the `Level.objects[...]` API return a `ReadResult` whose `positions` is one `(N, 3)` array and whose `parts` are slices into it: that is the same `data + offsets + lengths` triple that nibabel's `ArraySequence` holds and that `TrxFile.streamlines` already is. Second, `headers/<format>/` is an explicit, format-neutral place for a source header, so the TRX reference image survives without touching Zarr Vectors root metadata.

## Where the models disagree

Six differences need a decision; none is a blocker, and each has a clear answer.

**Vertex order.** TRX stores vertices in streamline order, so `positions[offsets[i]:offsets[i+1]]` is streamline `i` and `dpv` rows align by position. Zarr Vectors stores vertices in chunk order and reconstructs a streamline from its manifest. A backend therefore cannot hand out one memmap of the whole file. It hands out a `ReadResult` per request instead, which is already streamline-ordered, with `dpv` rows re-aligned in the same pass. Whole-file access becomes a sequential walk over chunks or objects, which Allen's own comparison rates at the same throughput as memmap.

**Dtype fidelity.** TRX accepts int8 through int64, uint8 through uint64, float16/32/64 and `bit`. The Zarr Vectors core encoder takes any numpy dtype, but `write_polylines` defaults positions to float32 and the `ingest_trx` converter widens float16 to float32, casts bool to uint8 and forces every `dpg` value to float32. A backend must write each array in the dtype TRX gave it and must round-trip `positions.3.float16` as float16. Whether the Zarr Vectors validator accepts float16 vertices is an open question for the Allen team.

**Booleans.** TRX `bit` arrays have no Zarr Vectors fill value, and the core refuses a bool array without an explicit `fill_value`. Proposed convention: store as `int8` with `fill_value=-1`, record the original dtype (`bit`) in an attribute on the array, and restore on read.

**Per-group data shape.** TRX `dpg` is one `(1, N)` array per group per key, and a group need not carry every key. Zarr Vectors `group_attributes/<name>` is one `(G,)` or `(G, C)` column across all groups, so a missing key becomes the column's fill value. The existing converter uses NaN and refuses keys whose width differs between groups. Proposed convention: keep NaN for floats and dtype-min/max for ints as the core already does, and record which groups actually carried the key in a `present` mask attribute so the backend can rebuild TRX's sparse `dpg` exactly.

**Affine and reference image.** TRX positions are RASmm by definition; `VOXEL_TO_RASMM` and `DIMENSIONS` describe the reference image and are not applied to positions. The backend writes positions as they are and stores the header verbatim in `headers/trx/`, which is what `ingest_trx` already does. The store's `axes` should declare millimetre units.

**Empty files and header counts.** TRX allows zero streamlines (the loader returns an empty `TrxFile`), while `write_polylines` raises on an empty list. The backend should create the store with `zv.create` and write nothing, so an empty TRX round-trips. `NB_STREAMLINES` and `NB_VERTICES` are not stored separately; they come from `object_index` and `vertex_count` on open, and the backend fills the header dict from them.

## Architecture

The change is one seam in trx-python: `TrxFile` stops assuming its arrays are memmaps and asks a backend for them. The public attributes (`header`, `streamlines`, `data_per_vertex`, `data_per_streamline`, `groups`, `data_per_group`) and the module functions (`load`, `save`, `concatenate`) keep their names and types, so DIPY, scilpy and existing scripts do not change.

```
Callers: DIPY, scilpy, MRtrix, user code
                │
                ▼
TrxFile (unchanged public API)
  header · streamlines (ArraySequence) · groups
  data_per_vertex · data_per_streamline · data_per_group
                │
                ▼
TrxBackend protocol (new, trx/backends/)
  open · header · counts · read(indices) · query(bbox) · write(trx) · close
        │                                   │
        ▼                                   ▼
MemmapBackend (today's code)        ZarrVectorsBackend (new)
  .trx zip or plain folder            .zarrvectors store (Zarr v3)
  np.memmap per array, whole file     Dataset, Level.objects, select(bbox)
  sequential and random by index      HeaderRegistry keeps header.json
        │                                   │
        ▼                                   ▼
Local disk                          Local dir, S3 / GCS, icechunk
```

A caller opens a path; `load` picks the backend from the container (zip or folder with `header.json` versus a Zarr v3 group with `zarr_vectors` attributes) and both backends present the same `TrxFile`.

**Backend protocol.** A small `Protocol` class in a new `trx/backends/` package, roughly:

```python
class TrxBackend(Protocol):
    header: dict                      # VOXEL_TO_RASMM, DIMENSIONS, NB_*
    def read(self, indices=None) -> TrxFile: ...     # streamline-ordered arrays
    def query(self, bbox) -> TrxFile: ...            # optional; memmap raises NotImplemented
    def groups(self) -> dict[str, np.ndarray]: ...
    def write(self, trx: TrxFile, path, **opts) -> None: ...
    def close(self) -> None: ...
```

The existing memmap code becomes `MemmapBackend` by moving `load_from_zip`, `load_from_directory`, `_create_trx_from_pointer` and `zip_from_folder` behind it; no logic changes. `TrxFile.select`, `get_group`, `append`, `resize` and the `to_sft` / `from_sft` converters keep operating on the in-memory arrays they get back.

**ZarrVectorsBackend on open.** `zv.open(path)` gives a `Dataset`; the backend checks `kind_of(0) == "streamline"`, reads `headers/trx/` into `header` (falling back to identity affine and `[1, 1, 1]` dimensions when absent), and fills `NB_STREAMLINES` from `level.objects.count` and `NB_VERTICES` from the level's `vertex_count`. Nothing else is read until asked. `groups()` comes from `level.groups` names and members; `dpg` from `group_attributes` filtered by the `present` mask.

**ZarrVectorsBackend on read.** `read(indices)` calls `level.objects[indices]` and receives a `ReadResult` with `positions`, `parts` and per-vertex `attributes` already streamline-ordered. The backend builds `ArraySequence` objects straight from `positions` and the `parts` slices (no copy), reads `object_attributes` rows for `dps`, and returns a `TrxFile` whose `header` counts match the subset. `read()` with no indices walks objects in batches and concatenates, which is the whole-file path. `query(bbox)` calls `level.select(bbox=...)` and is the new capability TRX users gain.

**ZarrVectorsBackend on write.** `write(trx, path)` creates the store with `zv.create` and a `Schema` whose bounds come from the positions, with millimetre axes and `kind="streamline"`, writes the positions in their TRX dtype, writes `dpv`/`dps`/`groups`/`dpg` with the dtype conventions above, stores `header.json` through `HeaderRegistry.add("trx", ...)`, and names group rows with `GroupCatalog.name_rows`. Chunk shape defaults to a size derived from `DIMENSIONS` and the affine voxel size (for example 32 voxels), overridable by the caller. Large files are written in object batches rather than a Python list of per-streamline arrays; this needs a streaming polyline writer in zarr-vectors-py (see the asks below).

**What stays lazy.** For the memmap backend, `TrxFile.streamlines._data` is still a memmap. For the Zarr backend, `TrxFile` holds no memmap; arrays are materialised per `read` or `query`. Code that relied on `_data` being a memmap (chiefly `resize` and the in-place `append` buffers) only runs on the memmap backend, and the Zarr backend raises a clear error for in-place growth, pointing to `write`.

## What each project changes

Most of the work lands in trx-python; the asks of zarr-vectors-py are small and mostly about guarantees rather than features.

| Project | Change | Size | Why |
| --- | --- | --- | --- |
| trx-python | `trx/backends/` package with `TrxBackend` protocol, `MemmapBackend` (moved code), `ZarrVectorsBackend` | Medium | The seam everything else hangs on |
| trx-python | `load()` detects a Zarr v3 group with `zarr_vectors` attributes; `save()` writes `.zarrvectors` when the extension or a `backend=` keyword says so | Small | Same entry points for users |
| trx-python | Optional dependency `trx-python[zarr]` pulling `zarr-vectors-py`; import guarded | Small | Core TRX stays dependency-light (numpy, nibabel) |
| trx-python | `TrxFile.query(bbox)` method, raising `NotImplementedError` on the memmap backend | Small | The new capability, surfaced in one place |
| trx-python | Round-trip test suite: every dtype in the TRX spec, sparse `dpg`, empty file, groups with names, `.trx` to `.zarrvectors` and back, byte-identical | Medium | Fidelity is the whole point |
| trx-python | `trx convert` CLI gains `.zarrvectors` as a target and source | Small | Users already use the CLI for TRK/TCK |
| zarr-vectors-py | Confirm float16 vertices and integer `vertex_attributes` pass validation at every level, or document the promotion | Small | TRX files default to float16 positions |
| zarr-vectors-py | A documented convention for bool arrays (`int8` with `fill_value=-1` plus an `original_dtype` attribute) | Small | TRX `bit` arrays |
| zarr-vectors-py | A `present` mask on `group_attributes` columns, or an agreed sentinel per dtype | Small | Rebuild TRX's sparse `dpg` exactly |
| zarr-vectors-py | Keep `Dataset`, `Level.objects`, `Level.select`, `ReadResult`, `HeaderRegistry` and `GroupCatalog.name_rows` in the stable `api` surface through 1.0 | Policy | trx-python can only depend on what will not move |
| zarr-vectors-py | Streaming polyline write that takes an iterator of `(positions, lengths)` batches instead of `list[np.ndarray]` | Medium | Multi-million-streamline TRX files must not be materialised per streamline |
| zarr-vectors-tools | Once the backend exists, `ingest_trx` / `export_trx` become thin wrappers over it, and the lossy `_stored_dtype` goes away | Small | One implementation of the mapping, not two |
| zarr-vectors-py docs | Correct the TRX comparison page (PR opened) | Small | Readers from the TRX community will check it first |

The `streamline` geometry type needs no change. The only format-level addition worth proposing upstream is a short note in `docs/spec/geometry_types/streamline.md` that `headers/trx/` is where a TRX reference image lives, so other Zarr Vectors readers know to look for it.

## Phased plan

The read path ships before any writer, because Allen-ingested stores already exist and a reader proves the mapping without committing either project to write conventions. Each phase has one gate; the next does not start until it passes. No dates are attached yet.

| Phase | Deliverable | Gate |
| --- | --- | --- |
| 0 · Agree the conventions (both teams) | Dtype fidelity, bool encoding, sparse dpg mask, `headers/trx/` location, stable api list | Conventions written into zarr-vectors-py docs and agreed in a tee-ar-ex issue |
| 1 · Backend seam in trx-python (pure refactor) | `TrxBackend` protocol; `MemmapBackend` wraps today's zip and folder code unchanged | Existing trx-python test suite green, no public API change |
| 2 · ZarrVectorsBackend read path | open, header, `read(indices)`, `query(bbox)`, groups and dpg; `trx-python[zarr]` extra | Reads Allen-ingested stores; positions, dpv, dps equal the source TRX |
| 3 · Write path and CLI | `save()` to `.zarrvectors`, streaming write in object batches, `trx convert` target | `.trx` → `.zarrvectors` → `.trx` byte-identical for every TRX dtype and sparse dpg |
| 4 · Converge and release | zarr-vectors-tools converters wrap the backend; comparison doc corrected | trx-python release with the zarr extra; zarr-vectors-py api surface pinned |

Phase 1 is deliberately a pure refactor so the TRX maintainers can review it on its own and the existing test suite is the acceptance criterion. Phase 4 is the point at which `zarr-vectors-tools` stops carrying its own mapping.

## Open questions

**For the TRX maintainers**

- Is a backend seam inside `trx_file_memmap.py` acceptable, or should the Zarr backend live in a separate package (`trx-zarr`) that registers itself with trx-python? A separate package keeps trx-python's dependency list untouched but splits the test suite.
- Should `TrxFile.query(bbox)` be part of the TRX Python API at all, given the memmap backend cannot implement it? The alternative is to expose it only on the backend object.
- For whole-file reads from a Zarr store, is a materialised in-memory `TrxFile` acceptable, or do downstream tools (scilpy's `--reference` workflows, DIPY's `StatefulTractogram`) need a lazy `ArraySequence`?
- Does the community want `.zarrvectors` mentioned in the TRX spec as an alternate container, or kept entirely at the implementation level for now?

**For the Allen Institute**

- Do float16 `vertices` and integer `vertex_attributes` pass `Dataset.validate()` at all four levels today? The core encoder accepts any dtype, but the writer defaults to float32 and the converter widens.
- Which of `Dataset`, `Level.objects`, `Level.select`, `ReadResult`, `HeaderRegistry`, `GroupCatalog` will be frozen through 1.0? trx-python would pin a minimum version and needs to know what is safe.
- Is a streaming polyline writer (iterator of batches) on the roadmap? `ZVWriter.append_vertices` is point-oriented and writes no cross-chunk bridges, so it is not a drop-in path for streamlines.
- Is there appetite for a shared conformance fixture set (small TRX files covering every dtype and sparse `dpg`) that both repositories test against?

## Appendix: code references

| Repository | What was read | Relevant to |
| --- | --- | --- |
| tee-ar-ex/trx-spec | `specifications.md` | Data model, dtypes, layout |
| tee-ar-ex/trx-python | `trx/trx_file_memmap.py`: `load`, `load_from_zip`, `load_from_directory`, `_create_trx_from_pointer`, `save`, `TrxFile.select`, `resize`, `append` | Where the backend seam goes |
| AllenInstitute/zarr-vectors-py | `README.md` store layout; `types/polylines.py` (`write_polylines`, `read_polylines`); `api/result.py` (`ReadResult`); `api/groups.py` (`GroupCatalog`, `name_rows`); `headers/registry.py`; `core/arrays.py` fill-value rules; `constants.py` format changelog; `docs/spec/geometry_types/streamline.md` | Mapping, dtype rules, stability |
| AllenInstitute/zarr-vectors-tools | `convert/ingest/trx.py`, `convert/export/trx.py`, `convert/export/_streamlines.py` | Existing converters and their dtype handling |
| AllenInstitute/zarr-vectors-py | `lazy/writer.py` (`ZVWriter.append_vertices`), `BRIDGE_APPEND_API_PLAN.md` | Streaming write options |

</details>
