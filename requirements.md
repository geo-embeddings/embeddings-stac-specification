# STAC Embedding Extension -- Requirements

- **Title:** Embedding
- **Field Name Prefix:** emb
- **Scope:** Item, Collection
- **Extension Maturity Classification:** Proposal
- **Related Specifications:**
  - [STAC Commons Metadata](https://github.com/radiantearth/stac-spec/blob/master/commons/common-metadata.md)
  - [STAC Collection Spec](https://github.com/radiantearth/stac-spec/blob/master/collection-spec/collection-spec.md)
  - [STAC MLM Extension](https://github.com/stac-extensions/mlm)
  - [GeoParquet](https://geoparquet.org/)
  - [GeoZarr](https://github.com/zarr-developers/geozarr-spec)

## Purpose

This document defines metadata requirements for a STAC extension describing geospatial embedding **products** -- the vector representations produced by running a geospatial foundation model against Earth observation data. These requirements will be serialized into JSON Schema for use as a STAC extension and also serve as metadata guidance for Zarr and GeoParquet stores.

This specification is a deliverable of the [Cloud-Native Geospatial Sprint](https://cloudnativegeo.org/blog/2026/01/announcing-sprints-to-define-best-practices-for-earth-observation-vector-embeddings/) at Clark University (March 10-11, 2026), convened by CNG, Planet, and Clark University to define best practices for EO vector embeddings.

### Relationship to STAC MLM

The [STAC MLM Extension](https://github.com/stac-extensions/mlm) describes the **model itself**: its architecture, inputs, outputs, and available parameters. This embedding extension describes a **specific execution** of a linked MLM-described model: what runtime parameters were used, what source data was fed in, and what pre/post-processing was applied to produce the embedding product. The two extensions are complementary -- MLM is the model card, this extension is the run receipt.

### Embedding Type Taxonomy

Geospatial embeddings in scope for this extension fall into two categories (per [arXiv:2601.13134](https://arxiv.org/abs/2601.13134)):

| Type | Description | Typical Format |
| --- | --- | --- |
| `pixel` | Dense, per-pixel embeddings preserving full spatial detail | COG, Zarr |
| `patch` | One embedding vector per spatial tile/chip | GeoParquet |

The embedding type drives storage format, resolution, and cost trade-offs. A `pixel` product at 10m resolution for Africa is ~38-77 TB; a `patch` product for the same area can be under 4 GB.

---

## Collection-Level Metadata

An overarching description of a corpus of embeddings generated with the same model and source data. For example: "All Sentinel-2-L2A observations encoded with the Prithvi model."

Builds on [STAC Commons Metadata](https://github.com/radiantearth/stac-spec/blob/master/commons/common-metadata.md) (omitting Instrument and Bands objects) and [STAC Collection Spec](https://github.com/radiantearth/stac-spec/blob/master/collection-spec/collection-spec.md).

### Collection Fields

| Field Name | Type | Required | Description |
| --- | --- | --- | --- |
| id | string | **Yes** | Unique identifier for this embedding collection |
| emb:version | string | **Yes** | Version of this embedding extension spec that the collection claims to implement |
| emb:type | string enum | **Yes** | Embedding type: `pixel` or `patch` |
| emb:dimensions | integer | **Yes** | Number of embedding dimensions (e.g., 64, 128, 768, 1024) |
| emb:dtype | string | **Yes** | Data type of stored embeddings (e.g., `float32`, `int8`, `uint16`) |
| emb:shape | \[integer] | No | Shape of a single embedding (e.g., `[768]` for patch, `[128, H, W]` for pixel) |
| emb:spatial_resolution | number | No | Ground sample distance or spatial extent per embedding, in meters |
| emb:patch_layout | Patch Layout Object | No | Patch extraction configuration for patch embeddings (regular or non-uniform grids, including overlap semantics) |
| emb:temporal_resolution | string | No | Temporal compositing window (e.g., `single_acquisition`, `annual`, `quarterly`, `multi_temporal`) |
| extent | Extent Object | **Yes** | Temporal and spatial extent (from STAC Collection spec) |
| license | string | **Yes** | SPDX license identifier (from STAC Collection spec) |
| assets | Assets Object | No | Collection-level assets |

### Patch Layout Object

This object records how patch embeddings were extracted. It supports both regular grids and non-uniform tiling schemes (e.g., grids that vary with latitude).

| Field Name | Type | Required | Description |
| --- | --- | --- | --- |
| layout_type | string enum | **Yes** | Patch layout strategy: `regular_grid`, `variable_grid`, or `named_grid` |
| patch_size | integer or \[integer] | No | Patch size in pixels (single value for square patches or `[height, width]`). SHOULD be provided for `regular_grid`; MAY be omitted when patch geometry is externally defined. |
| stride | integer or \[integer] | No | Step between neighboring patches in pixels. For `regular_grid`, if `stride < patch_size`, patches overlap. Not required for variable or externally defined grids. |
| grid_id | string | No | Identifier of the tiling/grid system used (e.g., a MajorTom grid name/version) |
| grid_definition | Link Object | No | Link to a formal grid specification, lookup table, or tiling definition used to generate patch footprints |
| patch_geometries | Link Object | No | Link to per-patch geometry definitions when patch footprints are non-uniform or cannot be inferred from `patch_size`/`stride` alone |

### Collection Links

The collection MUST include an array of links using the following relation types:

| Relation Type | Required | Description |
| --- | --- | --- |
| emb:model | **Yes** | Link to the encoder model. If the model is described as a STAC MLM Item, the link SHOULD point to that Item. |
| emb:decoder | No | Link to a decoder model, if one exists |
| emb:source-data | **Yes** | Link to the source data collection (e.g., Sentinel-2-L2A) |
| emb:paper | No | Link to the model or dataset paper |
| emb:benchmark | No | Link to an evaluation benchmark for the embeddings |

If the linked model is a STAC MLM Item, the collection SHOULD also reference which MLM input and output specifications were used for generating these embeddings.

---

## Item-Level Metadata

An item represents a single spatiotemporal partition of the collection -- a tile, a time step, or a region. It is the unit at which runtime execution metadata is recorded.

### Item Fields

| Field Name | Type | Required | Description |
| --- | --- | --- | --- |
| emb:runtime_parameters | object | No | The specific parameter values used for this inference run (e.g., input bands, patch size, stride, temporal window). Structure is model-dependent. |
| emb:inference_datetime | string (date-time) | No | When the embeddings were generated |
| emb:source_data | \[Link Object] | No | Links to the specific source imagery items/scenes used to produce this item's embeddings |
| emb:source_data_hash | string | No | Hash of the source data for reproducibility verification |
| emb:processing_baseline | string | No | Source data processing baseline version (e.g., Sentinel-2 processing baseline) |
| emb:preprocessing | Processing Object | No | How source data was prepared before model inference. SHOULD reuse the [MLM processing expression](https://github.com/stac-extensions/mlm) format. |
| emb:postprocessing | Processing Object | No | Transformations applied to raw model output (dimension reduction, normalization, quantization pipeline). SHOULD reuse the MLM processing expression format. |

### Reproducibility

The item-level metadata above serves a reproducibility goal: given a collection (which links to the model via MLM) and an item (which records the runtime parameters and source data), there should be enough information to regenerate the embeddings from scratch.

---

## Asset-Level Metadata

An asset is the actual data file: a Zarr store, a GeoParquet file, a COG, etc. Asset-level metadata describes the physical encoding and access properties of the file.

### Asset Fields

| Field Name | Type | Required | Description |
| --- | --- | --- | --- |
| emb:quantization | Quantization Object | No | Quantization details, if the embeddings have been quantized from their original dtype |
| emb:crs | string | No | Coordinate reference system of the asset (e.g., `EPSG:4326`). CRS SHOULD be embedded in the file itself, not only in metadata. |

### Quantization Object

If embeddings have been quantized (e.g., from `float32` to `int8`), the quantization object provides the information needed to interpret or dequantize them.

| Field Name | Type | Required | Description |
| --- | --- | --- | --- |
| method | string | **Yes** | Quantization method: `none`, `linear`, `sqrt_scale`, `quantization_aware_training`, or other |
| original_dtype | string | **Yes** | The data type before quantization (e.g., `float32`) |
| scale | number or \[number] | No | Dequantization scale factor(s) |
| offset | number or \[number] | No | Dequantization offset(s) |

---

## Discovery Metadata

These fields support finding and evaluating embedding products. They may appear at Collection or Item level.

| Field Name | Type | Required | Description |
| --- | --- | --- | --- |
| emb:fit_for_use | \[string] | No | Intended use cases (e.g., `classification`, `change_detection`, `retrieval`, `segmentation`, `similarity_search`) |
| emb:spatial_coverage | string | No | Geographic scope (e.g., `global`, `continental`, `regional`, `local`) |

Name, description, and license are already covered by STAC Commons Metadata and the Collection spec and SHOULD NOT be duplicated under the `emb:` prefix.

---

## Uncertainty Metadata

Uncertainty quantification for embeddings is an emerging area. The extension provides a lightweight sub-schema for recording it when available.

| Field Name | Type | Required | Description |
| --- | --- | --- | --- |
| emb:uncertainty.method | string | No | How uncertainty is quantified (e.g., `ensemble_variance`, `mc_dropout`, `calibration_score`) |
| emb:uncertainty.description | string | No | Human-readable description of uncertainty characteristics or limitations |
| emb:uncertainty.asset | Link Object | No | Link to a companion uncertainty data asset |

---

## Storage Format Recommendations (Non-Normative)

This section is informational and does not define normative requirements.

- **Patch embeddings** are best stored in **GeoParquet** -- one geometry + one embedding array per row. This aligns with how Earth Index, Clay, and Major TOM distribute their products.
- **Pixel embeddings** are best stored in **COG** (Cloud Optimized GeoTIFF) or **Zarr** -- raster format with embedding dimensions as bands or array variables. This is how AlphaEarth and Tessera distribute products.
- **CRS MUST be embedded in every data file**, not only documented in README files or external metadata. Implicit CRS assumptions are a major source of integration bugs across the ecosystem.
- **Cloud-native formats** (COG, GeoParquet, GeoZarr) should be treated as a baseline requirement for new products to enable efficient range-request access without full downloads.
