# Code review notes

Review of branch `add-region-mode` at commit `2a51274`, 2026-11-06.

This is based on reading `main.rs`, `cli.rs`, `read_parser.rs`, `region_parser.rs`, `binning.rs`, `variant.rs`, `reader.rs` and `errors.rs`. `features.rs` was only skimmed, and in `output.rs` only the output-path and merge functions were read. Items #6 and #7 were added after the first pass, so they are numbered after the other items to keep earlier references stable. `docs/REGION_IDEAS.md` has the region-mode design discussion. Items are marked **verified** where the code was read and the conclusion follows directly from it, and **unverified** otherwise. No changes have been made to the code.

## Bugs

### 1. BED parsing drops the last two bases of every region — FIXED (not yet committed)

**Where:** `src/region_parser.rs`, `parse_bed` and `process_region_row`

**Status:** fixed in the working tree on top of `17c0044`. `parse_bed` now converts with `start = s + 1` and `end = e + 1`, so `RegionRow.end` is a 1-based *exclusive* end. `row.start..row.end` then yields exactly `s + 1 ..= e`, and `end > start` is the same check as `e > s` on the raw BED values. Unit tests in `region_parser::tests` cover this; run them with `cargo test region_parser`:

- **`parse_bed_coordinate_cases`:** `0 10` → 1–10, 1-base and 2-base regions, extra BED columns, empty and reversed regions, and an end of `0` (no underflow).
- **`parse_bed_invalid_row_cases`:** too few columns, non-numeric or negative coordinates.
- **`parse_bed_region_positions_have_no_alleles`:** chrom and pos are set, with no ref/alt and `VarType::Unknown`.

**Leftovers (cosmetic, not bugs):**

- `start > 0 && end > 0` is now always true; only `end > start` matters.
- `RegionRow.end` needs a comment saying it is an exclusive end, so it isn't "fixed" back to inclusive.
- The skip warning text ("Start/end pos is > 0 or end pos > start pos") says the opposite of the condition; something like "region is empty (end <= start)" would be clearer.

**Original problem** (commit `2a51274`): the code used `end = e - 1` with the exclusive loop `row.start..row.end`. For `chr1 0 10` that gave positions 1–8. Regions of 1 or 2 bases were rejected, and an end of `0` underflowed the `u64` (a panic in debug builds, a wrap-around in release). Commit `17c0044` changed this to `end = e`, which removed the underflow but still dropped the last base.

### 2. The "CRAM requires FASTA" check can never fire (verified)

**Where:** `src/read_parser.rs`, `open_indexed_reader`

`fasta_path.ok_or(AppError::MissingCramReference)` sits inside `if fasta_path.is_some()`, so the error is never raised. A CRAM opened without `--fasta` goes straight to htslib's fallback reference lookup. No extra code or parameter is needed for that lookup to happen.

#### How htslib finds a reference

Source: htslib 1.19.1, bundled in `hts-sys` 2.2.0 (`cram/cram_io.c`, `cram_get_ref` and `cram_populate_ref`). This matches the REF_PATH/REF_CACHE section of the samtools(1) manual.

When a CRAM slice needs reference bases for a contig, htslib tries these in order:

1. **An explicit FASTA (`--fasta`).** rust-htslib's `set_reference` calls `hts_set_fai_filename`. If a contig is missing from that FASTA, htslib warns `Reference file given, but ref '…' not present` and moves on to the steps below.
2. **The `M5` checksum in the CRAM header.** Each contig's `@SQ` line usually has an `M5` tag, the MD5 of that contig's sequence. htslib uses it to look in:
   - **`REF_CACHE`**, a local cache directory.
   - **`REF_PATH`**, a list of local directories or URLs.
   - **If `REF_PATH` is unset**, it defaults to the EBI server `https://www.ebi.ac.uk/ena/cram/md5/%s`. In that case htslib always sets up a local cache at `$XDG_CACHE_HOME/hts-ref`, or `~/.cache/hts-ref` if that's unset (falling back to `$TMPDIR`, then `/tmp`). Downloaded sequences are checked against the MD5 and saved to the cache.
3. **The `UR` tag in the CRAM header.** If there's no `M5`, or the M5 lookup fails, htslib tries the file path in `@SQ UR:` and loads it with its `.fai` index.
4. **Failure.** If all of that fails, htslib logs `Failed to populate reference for id N` and decoding fails.

This build includes the EBI download. rust-htslib's default features turn on `curl`, and `cargo tree -e features -i hts-sys` confirms `hts-sys/curl` is enabled. That explains the `~/.cache/hts-ref` cleanup step in `test_data/notes.md`.

CRAMs that embed their reference, or are written without needing one, need no lookup at all. That code path was not traced.

#### Recommendation

- **Keep `--fasta` optional.** Making it required would block CRAMs that resolve fine through `M5`/`REF_PATH`. Delete the unreachable `ok_or(AppError::MissingCramReference)` (and the `MissingCramReference` variant if nothing else uses it). Update the README and the `--fasta` help text, which both say FASTA is required.
- **Make the network fallback visible.** With no `--fasta` and no `REF_PATH`, the tool silently contacts EBI and may download whole chromosomes into the home directory; htslib's own warning says the cache "may become large". On HPC compute nodes without internet, the lookup just fails. Two fixes:
  - Document `REF_PATH`/`REF_CACHE` in the README.
  - Log one `warn!` when a CRAM is opened without `--fasta`, saying htslib will resolve references through `REF_PATH`/`REF_CACHE`/EBI.
- **Check the error message for a missing reference (unverified).** If no reference can be found, each `reader.read()` probably returns an error. After 10 in a row, `process_bin` would report `TruncatedReads` ("truncated or corrupted reads file"), which is misleading. To test: move `~/.cache/hts-ref` aside, unset `REF_PATH`, and run on a CRAM offline or with a fake `REF_PATH`. If confirmed, make the error mention a missing reference as a possible cause for CRAM input.

### 6. `--merge` with one reads file deletes the output (verified by reading; not run)

**Where:** `src/output.rs`, `make_output_csv_paths` and `merge_output_csvs`

With a single reads file, `make_output_csv_paths` returns the `--output` path itself as the per-file CSV. `merge_output_csvs` then:

1. Opens that path for reading. The reader is lazy, so nothing has been read yet.
2. Calls `File::create` on the same path, which truncates it to zero bytes.
3. Reads an empty header and no rows, and writes an empty merged file.
4. Deletes every input path, which is the output.

The run reports success and leaves no output CSV.

**Fix:** in `main.rs`, skip the merge when `common.reads.len() == 1`, or return early from `merge_output_csvs` when there is a single input. Also refuse to merge if `output_path` is one of the `input_paths`.

### 7. Merge doesn't check that rows line up (verified)

**Where:** `src/output.rs`, `merge_output_csvs`

Rows are paired by line number only. For the second and later files, the `bool` returned by `read_byte_record` is ignored, so a shorter file is not detected and the record buffer is reused silently. `chrom`/`pos` are never compared. Every per-file CSV is generated from the same variant list in the same order, so this is low risk today. It would hide a future bug, though, such as a partial file or a change in ordering.

**Fix:** error if any later file ends before the first file, or the first file ends before any later one. Check that the `chrom` and `pos` fields match across files on every row. Both checks are cheap.

## Fragile by design

### 3. Everything assumes sorted input (partly verified)

`read_parser::process_bin` writes a variant as soon as a read starts past it. If the input is out of order, rows are written early with too little coverage and no error is raised.

- **Variants:** `variant_parser` checks sorting (`AppError::UnsortedVariants`).
- **Regions:** `region_parser` has no sorting check. Overlapping or unsorted BED regions produce positions that go backwards. These probably hit the distance check in `binning::get_variant_distance`, but this is **unverified**. Overlapping regions would also produce duplicate positions.
- **Reads:** safe, because indexed `fetch` returns sorted reads.

### 4. Region mode's memory use (known)

`region_parser` creates one `VariantInfo` per base, each with its own cloned chromosome `String`, all before any reads are scanned. A whole-exome BED means tens of millions of records.

**Direction:** expand regions one bin at a time (an iterator, or a parser that emits bins), and share chromosome names (`Arc<str>` or an index into a chromosome table) instead of cloning them.

### 5. `main.rs` has the pipeline twice (verified)

The `Mode::Loci` and `Mode::Region` branches are nearly identical, differing only in the parse call and how the variant class is chosen. Any fix to one has to be repeated in the other. The region branch also still has two debug `println!` lines at the end.

**Fix:** pull out one function `run_pipeline(parsed: &ParsedVariants, varclass: VarClass, common: &CommonArgs)` and call it from both branches.

## Smaller items

- **Exit codes:** the comment in `errors.rs` reserves 2 for command-line errors, but no `AppError` uses it (clap exits with 2 on its own). Every non-`AppError` failure exits with 1, which `main.rs` marks "FIX LATER".
- **Truncation detection:** only fires after 10 read errors in a row (`MAX_CONSECUTIVE_ERRORS`). Scattered corruption, or a missing EOF block, produces warnings and an output that looks valid.
- **Test coverage:** the unit tests cover `variant_parser`, `binning` and BED parsing in `region_parser`. There are none for `read_parser`, `features` or `output`, and `tests/integration_test.rs` is empty. The chrM parity check (`tests/compare_outputs.sh`) is the only end-to-end test, and it is run by hand.

## Areas not yet reviewed

- **`features.rs` indel handling:** `LocusFeaturesIndel::count`, `indels_overlapping_variant` and `get_indel_start_coord` are about 200 lines of interval logic. This is where off-by-one errors and differences from the Python output are most likely. The chrM fixture only has SNV output (`python.chrM_heavy_stress.snv.csv`). An indel parity fixture would add the most confidence.
- **`--label` vs `--reads` count:** not checked whether a mismatch is caught (possibly in `make_output_csv_paths`).

## Suggested order

1. Commit the #1 fix and its tests. Fix #6 (`--merge` deletes single-file output) and add a unit test for it. Add the #7 checks at the same time.
2. #2: remove the dead CRAM check, document `REF_PATH`/`REF_CACHE`, add the "no `--fasta`" warning, and test the missing-reference error message.
3. Fix #5 (one shared pipeline function) before starting #4, so the batching work only touches one path.
4. #4: batched region expansion.
5. Add an indel parity fixture, then wire the chrM parity check into `tests/integration_test.rs` so `cargo test` runs it.
