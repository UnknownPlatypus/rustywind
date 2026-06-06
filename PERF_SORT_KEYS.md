# Tailwind class-sort performance work

This branch (`perf/precompute-sort-keys`) makes `rustywind_core`'s pattern-based
Tailwind class sorting dramatically faster without changing its output.

## Why

Integrating this sorter into [djangofmt](https://github.com/UnknownPlatypus/djangofmt)
as the `unsorted-tailwind-classes` lint rule
([PR #332](https://github.com/UnknownPlatypus/djangofmt/pull/332)) caused a large
linter-benchmark regression: **~49% overall**, up to **−74%** on the worst file
(`comparison_table_integrated.html`, 4.6 ms → 17.8 ms). The rule calls a shared
`RustyWind` → `HybridSorter` once per `class` attribute, so all the cost is inside
`rustywind_core`'s sort.

## Root cause

Two cost centers, both confirmed by microbenchmarks:

1. **`SortKey::cmp` re-derived everything on every comparison.** Sorting does
   `O(n·log n)` comparisons, and each one rebuilt the variant mask
   (clone + sort + dedup), re-mapped arbitrary-variant keys, and re-ran
   `extract_color_name` / `has_opacity_syntax` / `extract_base_number` /
   `should_arbitrary_come_first` / `extract_base_name` /
   `get_utility_prefix_priority` on the raw class string. Several of those
   allocate. A single `p-4` vs `p-8` comparison took **94.5 ns**.
2. **Unrecognized (custom) classes were never cached.** `HybridSorter::get_sort_key`
   only inserted *recognized* utilities; a custom class such as `icon-check` returned
   `None` without being cached, so every occurrence re-ran `parse_class` +
   `get_properties`. Templates that repeat custom classes paid that on every element.

The fix follows the pattern ruff's isort uses for the same problem (sorting arbitrary
identifiers with no perf issue): **decorate-sort-undecorate** — build a precomputed,
cheap-to-compare key once per item, then sort on the keys.

## Changes (4 commits)

| SHA | Commit | What it does |
|-----|--------|--------------|
| `fd9b80f` | Look up variant order via a hash map instead of linear scans | `get_variant_index` ran up to ~14 linear `.position()` scans over the 82-entry `VARIANT_ORDER` per call (and it is called many times per comparison). Replaced with a `LazyLock` hash map for exact matches, keeping prefix fallbacks. |
| `bd0bb8c` | Precompute sort key fields so comparisons never re-derive | Moved every comparison input into `get_sort_key` and stored it on `SortKey` (`variant_mask`, `selector_dynamic_seq`, `arbitrary_variant_keys`, `utility_prefix_priority`, `color_name`, `has_arbitrary_value`, `has_opacity_syntax`, `arbitrary_comes_first`, `base_number`, `base_name`). `cmp` now only reads fields — no allocation, no string re-parsing. |
| `134b97b` | Cache sort keys behind `Arc` so hits and sort swaps stay cheap | The sort stored `SortKey` by value, so every cache hit deep-cloned it (now several `Vec`s) and every sort swap moved the whole struct. The cache now holds `Arc<SortKey>`: a hit is a refcount bump and a swap moves a pointer. |
| `ecd0a8b` | Cache unrecognized classes so repeats aren't re-derived | Cache value is now `Option<Arc<SortKey>>`, so a custom class is parsed once and then served from the cache instead of re-derived on every occurrence. |

Behaviour is unchanged: the full `rustywind-core` test suite (144 unit tests +
~25 ordering / fuzz-regression integration files) passes, and the optimized binary
produces **byte-identical output to the baseline across all 2402 real templates** in
`~/greenday/mysite` (see below).

## Microbenchmarks

`cargo bench -p rustywind_core --bench sorter_benchmark`, release profile
(`lto = "fat"`, `codegen-units = 1`, `opt-level = 3`), single Linux machine, criterion.

### Cumulative — all 4 commits vs the pre-fix baseline

| Benchmark | Before | After | Change |
|-----------|-------:|------:|:------:|
| `compare_numeric` (single `SortKey::cmp`) | 94.5 ns | 9.70 ns | **−89.8%** |
| `compare_property` | 11.5 ns | 3.67 ns | **−68.2%** |
| `compare_variant` | 1.84 ns | 1.68 ns | −8.6% |
| `sort_classes/small_10` | ~0.98 µs | 0.377 µs | **−61.6%** |
| `sort_classes/medium_25` | ~4.0 µs | 1.35 µs | **−66.2%** |
| `sort_classes/large_80` | 26.5 µs | 6.81 µs | **−74.4%** |
| `sort_classes/variant_heavy_25` | 13.3 µs | 2.56 µs | **−80.8%** |
| `sort_classes/custom_heavy_20`¹ | 4.75 µs | 0.619 µs | **−87.0%** |
| `cache_performance/warm_cache` | 26.3 µs | 6.68 µs | **−74.8%** |
| `cache_performance/cold_cache` | 74.5 µs | 64.6 µs | −13.8% |
| `get_sort_key_single`² | 60.0 ns | 92.4 ns | +53.9% |
| `get_sort_key_compound_variant`² | 79.5 ns | 134 ns | +68.6% |
| `get_sort_key_arbitrary`² | 49.1 ns | 63.7 ns | +29.7% |

¹ Benchmark added in `ecd0a8b`; "before" measured on the preceding commit.
² Per-key generation got heavier because work moved up front — but keygen is cached
once per unique class, which is why the all-keygen `cold_cache` path is still net −14%
and the warm path is −75%.

### Per-commit contribution

| Commit | Headline results |
|--------|------------------|
| `fd9b80f` (hash-map lookup) | `variant_heavy_25` −49.0%, `medium_25` −4.4%, `cold_cache` −4.7% |
| `bd0bb8c` (precompute) | `compare_numeric` −89.5%, `compare_property` −68.5%, `large_80` −40.8%, `warm_cache` −41.6% |
| `134b97b` (Arc cache) | `small_10` −63.3%, `medium_25` −60.9%, `large_80` −56.3%, `variant_heavy_25` −60.4%, `warm_cache` −56.7% |
| `ecd0a8b` (negative cache) | `custom_heavy_20` −87.0% (4.75 µs → 619 ns); recognized-class benches unchanged |

## Real-world smoke test (`~/greenday/mysite`)

2402 HTML templates, ~28.8k `class="…"` attributes (almost all custom classes).
Baseline binary built from `master` (`3c50e91`, the exact rev djangofmt pins);
optimized from this branch.

**Correctness:** ran each binary in `--write` mode over a private copy of the tree
and `diff -r`'d the results → **byte-identical output on all 2402 templates**.

**Performance** (`hyperfine`, 3 warmup runs):

| Scenario | Baseline | Optimized | Result |
|----------|---------:|----------:|:------:|
| Full CLI `--check-formatted` (walks the tree — **I/O-bound**) | 129.4 ms wall / 178.7 ms user CPU | 129.6 ms wall / 145.1 ms user CPU | wall flat, **user CPU −19%** |
| Pure sort, `--stdin` on a 7 MB concatenated blob (one cached read — **CPU-bound**) | 67.0 ms / 54.8 ms user | 38.5 ms / 26.8 ms user | **1.74× faster, user CPU −51%** |

The CLI is dominated by filesystem I/O (System time ~290 ms), so wall-clock doesn't
move even though the sort burns 19% less CPU. Isolating the sort itself — which is what
the djangofmt lint rule does on already-parsed, in-memory templates, and what CodSpeed
measures in instructions rather than wall-time — it is **1.74× faster**.

## Reference: the ruff isort pattern this mirrors

[`crates/ruff_linter/src/rules/isort/sorting.rs`](https://github.com/astral-sh/ruff/blob/main/crates/ruff_linter/src/rules/isort/sorting.rs)
defines `ModuleKey` / `MemberKey`: `#[derive(Ord)]` key structs whose fields are all
precomputed once (expensive bits guarded behind `Option`), and
[`order.rs`](https://github.com/astral-sh/ruff/blob/main/crates/ruff_linter/src/rules/isort/order.rs)
sorts them with `Itertools::sorted_by_cached_key` (compute the key once, sort on the
cached keys, never recompute per comparison). That is exactly the shape of `bd0bb8c`
(precompute the key) + `134b97b` (keep the stored key cheap to move). `fd9b80f`
(O(1) name lookup instead of a scan) and `ecd0a8b` (memoize misses, not just hits) are
the general optimizations that make the per-item key generation cheap enough to cache.

## Reproduce

```sh
cargo bench -p rustywind_core --bench sorter_benchmark            # microbenchmarks
cargo test --workspace                                            # correctness

# real-world (no writes to your files):
cargo build --release -p rustywind
find ~/greenday/mysite -name '*.html' -exec cat {} + > /tmp/all.html
hyperfine --warmup 3 'target/release/rustywind --stdin < /tmp/all.html > /dev/null'
```
