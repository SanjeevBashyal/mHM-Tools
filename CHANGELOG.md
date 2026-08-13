# Changelog

## [v0.2.2]

### Added

- Always derive `L1_soilMoist` in the merged restart as the midpoint between `L1_wiltingPoint` and `L1_soilMoistFC` when both are available, overwriting any value a native tile restart may already carry.

### Changed

- Extend `generate_bounds`/`generate_bounds_for_all_coords` to accept an explicit resolution fallback for coordinates with only one point, where a cell width can no longer be derived by differencing.
- Add a `create_bounds` option to `get_dataset_from_path`, mirroring the existing option on `get_xarray_ds_from_file`, to attach real coordinate bounds to a dataset at open time, before any cropping.
- Move `regrid_mask` from `mhm_tools.pre.crop_mhm_setup` to `mhm_tools.common.xarray_utils`, since it's pure xarray grid-matching logic already used by both pre- and post-processing modules.
- `write_mask_file` now writes the catchment mask once (as `mask`) and adds `land_mask` as an HDF5 hard link to the same on-disk data via the new `add_variable_hard_link` helper (`mhm_tools.common.netcdf`), instead of writing the identical array twice under both names. Both variables remain fully and independently readable by any NetCDF/HDF5 tool, including all attributes, with no duplicated storage.
- `write_metric_plots` now always writes the same per-variable/per-realisation statistics table (n, min, max, mean, median) to `metric_summary.csv` via the new `write_metric_summary_csv`, independent of which `--plot-types` were requested, so the numbers are available as a machine-readable file rather than only inside the optional overview PDF.
- `write_metric_plots`'s `metric_plots_overview.pdf` is now written whenever there is at least one plot or summary row to show, instead of being skipped for the common case of a single variable and a single plot type; it now also embeds every plot type actually produced in that call (previously it could end up containing only catchment-map plots).

### Fixed

- Fix `cut_to_filled_area` producing an empty crop, and silently dropping the last filled row/column even when multiple cells are filled, due to an off-by-one in its bounding-box-to-slice conversion. A single-cell spatial mask previously crashed `gridded-data-evaluation` with `Cannot determine file resolution: no valid lon or lat coordinates provided`.
- Fix `generate_bounds_for_all_coords` writing the `bounds` attribute onto a discarded copy of the dataset instead of the one it returns, so the generated `*_bnds` coordinate existed but nothing pointing to it.
- Keep real CF coordinate bounds on `gridded-data-evaluation` statistics through cropping, including crops down to a single cell, instead of relying on resolution being re-derivable from the already-cropped coordinates afterward.
- Fix `regrid_mask` crashing with `IndexError` when a target or mask coordinate had been cropped to a single point: it now honors a caller-supplied `target_res`/`mask_res` instead of always re-deriving them from the raw coordinate arrays, and its own fallback now derives resolution via `get_file_res` (trying both lon and lat) instead of assuming `lon` has at least two points.
- Fix `get_file_res` returning a negative resolution for descending coordinates, which also silently broke matching against configured `l0`/`l1`/`l2`/`l11` resolutions for such files.
- Fix `create-mhm-restart-from-setup` merge producing an mHM-unreadable restart file: soil-horizon, land-cover-period, and LAI-timestep boundaries were looked up under invented native variable names, and a missing `return` corrupted the default six-horizon boundaries — both silently replaced real depths/periods/months with meaningless index values that mHM rejected on restart.
- Fix border-clobbering when merging restart tiles shared across multiple mask/parameter runs: a later tile's fill/NaN cells could overwrite an earlier tile's valid data at the same position.
- Mask each tile restart file in place with its own tile mask before it is moved or merged, so unmasked edge values can no longer leak into relocated or merged restart output.
- Write a tile's mask section on demand when reusing an existing tile setup (`--no-tile-creation`) whose `mask_tile.nc` predates this file, instead of requiring the whole tile to be recreated.
- crop_mhm_setup tryed to read mask regardless of whether it was provided or not
- avoid division by zero in SPEAF by returning nan whenever std of one array is zero
- Fix `resample_to_target_freq` silently dropping all data before the final overlap check: its closing `xr.align(..., join="inner")` joined on every shared dimension, so `lat`/`lon` labels that were already positionally matched by earlier cropping/regridding but still differed by float32-vs-float64 precision (e.g. `68.1500015258789` vs `68.15`) failed an exact-equality join and collapsed to near-empty, leaving `gridded-data-evaluation` unable to find any temporal overlap between input and reference.

### Removed

- Remove the unused `merge_mhm_restart_files` restart-merge pipeline and the ~15 helper functions reachable only from it (superseded in production by `merge_restart_files`), plus dead entries in the internal variable/dimension rename lookup tables that never matched real mHM restart output.

### Tests

- Add direct unit coverage for `cut_to_filled_area`: single-cell crops, inclusion of the last filled row/column, buffer clipping at both array bounds, and an upscaling edge case at the array boundary.
- Add coverage for `create_bounds` on `get_xarray_ds_from_file`/`get_dataset_from_path`, including that the coordinate's `bounds` attribute is set correctly.
- Rewrite `gridded-data-evaluation` single-point-crop regression tests to check for real coordinate bounds rather than a cached resolution attribute.
- Add `regrid_mask` coverage (moved to `tests/test_xarray_utils.py`) for resolution-fallback correctness: falling back across axes when a coordinate has a single point, honoring an explicitly provided resolution over a diffed one, and single-cell domains with a resolution supplied directly or derived from CF bounds.
- Add regression coverage for `resample_to_target_freq` confirming that lat/lon labels differing only by float32-vs-float64 precision no longer collapse the aligned output, using coordinate arrays reproducing the exact mismatch (`68.15` vs `68.1500015258789`).
- Add coverage for `write_mask_file` confirming `land_mask` and `mask` stay independently readable with identical data while verifying, via direct HDF5 object-address inspection, that `land_mask` is a genuine hard link rather than a separately stored duplicate.

## [v0.2.1]

### Fixed

- Calculate hydrograph KGE/NSE from the cropped overlapping discharge period instead of stale pre-crop arrays.
- Keep hydrograph objective and catchment state per `Hydrograph` instance to avoid stale metrics leaking between runs.
- Limit mHM restart tile-mask discovery to the restart output folder and owning tile folder to avoid unrelated parent masks affecting merges.
- Apply gridded ESP and SPAEF-like metrics per timestep before averaging, instead of flattening the full time-space array first.
- Handle single-point temporal overlaps in xarray utilities and improve the related crop error logging.
- Fix `create_header()` output handling for explicit file paths, missing parent directories, and existing directories with dots in the name.
- Prevent file output helpers from replacing existing files when the requested output path has no file suffix.
- Handle dotted gauge output directories in catchment creation.
- Use lon/lat box resolution as fallback for L0 resolution in `create-catchment`.
- Added ("longitude", "latitude") to possible xy coordinates in discharge file

### Changed

- Write catchment gauge-correction `score`, `shape_error`, and `method` columns to the gauge info CSV.
- Refactor NetCDF writing into `write_xarray_to_netcdf()` and shared helpers in `mhm_tools.common.netcdf`.
- Update install instructions in the README.

### Tests

- Add catchment gauge info CSV coverage for correction score, shape error, and method metadata.
- Add regression coverage for hydrograph KGE after cropping.
- Add `create_header()` path handling coverage, including CLI `--only-header` output to an explicit file.
- Add and update NetCDF encoding tests for the refactored NetCDF helper functions.
- Update spatial metric tests for corrected ESP/SPAEF output names and timestep-wise behavior.
- Add xarray overlap regression coverage.

## [v0.2]

### Added

- Initial official release for mHM 5 pre-processing, post-processing, and evaluation workflows.
- Add `create-catchment` to delineate basins from DEM or flow-direction data, correct gauge outlets by area or shape, and write mHM/mRM basin, mask, idgauges, and gauge metadata outputs.
- Add `crop-mhm-setup` to crop existing mHM setups to domains, masks, or bounding boxes while preserving required grid files and headers.
- Add `create-header` and `latlon` tools to generate mHM-compatible ASCII headers and lon/lat NetCDF grids from setup extents and resolutions.
- Add `prepare-mhm-forcings` and `calculate-pet` to prepare meteorological forcings, normalize units, handle temporal frequency, and derive PET fields.
- Add `create-mhm-restart-file` and `create-mhm-restart-from-setup` to build restart files from target grids or tiled setup runs, including masking and merge support.
- Add `create-subdomain-masks`, `create-idgauges`, and region/catchment masking helpers for domain partitioning and routing setup preparation.
- Add data-processing tools for file conversion, merging many files, regridding, filling missing values, calculating long-term means, ratios, differences, and relative differences.
- Add `discharge-evaluation` to compare observed and simulated discharge, match gauges, calculate metrics, and create CDF and map outputs.
- Add `hydrograph` to read discharge series, calculate objective metrics, and create hydrograph, seasonality, scatter, and flow-duration plots.
- Add `gridded-data-evaluation` for spatial and temporal comparison of gridded model outputs with metrics such as ESP, SPAEF, MSPAEF, and WASPAEF.
- Add `mhm-run-overview`, `2d-map`, and `taylor-diagram` tools for run summaries and visualization.
- Add utility commands such as `link-folder-tree` and initial mHM 5 to mHM 6 land-cover ASCII-to-NetCDF conversion support.

### Changed

- Switch to the Click-based CLI with grouped commands, aliases, typo suggestions, and optional Trogon support.
- Improve NetCDF metadata, coordinate handling, mask handling, and output provenance across generated files.

### Fixed

- Stabilize catchment shape and area correction, gridded evaluation masking, restart creation from setup tiles, hydrograph reading, and header generation.