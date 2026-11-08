Your plan, as your notes record it
Threading: one thread per reads file, with the remaining threads split evenly across files. Each sample’s bins would be chunked greedily across its sub-threads. You were still weighing std::thread::scope against rayon.
Output: it had to stay in sorted order. You thought one CSV per bin would cost too much I/O.
Batching: you wanted at most 16 batches in total across all samples (16 for 1 BAM, 8 for 2, and so on). Each sample’s bins would be split into batches balanced by total variant count, using an optimising split rather than a simple cut. You hadn’t decided between in-memory chunks and temp files, and you were worried about peak memory if all samples finish at once.
Region mode: a full chr12 BED was OOM-killed at about 32 GB before the binning step.
What I found in the code
Region mode memory: it expands every BED base into its own VariantInfo, which is roughly 120 bytes each. Binning then creates a much larger Variant per position, because adjacent positions are 1 bp apart and the whole chromosome becomes one bin. That explains the OOM.
Duplicate bins: each sample calls bin() itself, so every sample holds its own full copy of the variants.
Bin chunking: a germline VCF with the 100 kb gap gives about one bin per chromosome, so chunking whole bins caps parallelism at around 24 units.
Bugs I flagged:
BED conversion: it is off by two, so each region loses its last two bases.
BED end of 0: an end of 0 can wrap and run away.
--merge: with a single reads file, --merge truncates and then deletes the file it is reading.
Merge alignment: the merge doesn’t check that rows line up across files.
What I suggested
Region mode: keep regions as intervals and tile them into windows of 100 kb to 1 Mb, with a dense accumulator array per window. Region mode never uses ref or alt, so its accumulator can be smaller.
Binning: bin once and share the spans as index ranges. Allocate accumulators in the worker and drop them after writing.
Threading: chunk at variant boundaries and balance by span, not variant count. Replace the 16-batch cap with a work queue that has an ordered writer, using std::thread::scope. Measure IndexedReader::set_threads first.
Ordered writer:
Workers claim chunk indexes in order and serialize rows to a Vec<u8>.
One writer thread keeps a BTreeMap of pending chunks and writes them in index order.
A window gate (about 2 to 4 times the worker count) bounds memory.
It writes to a .partial file, then renames it.
Versus batches and temp files: the ordered writer writes each byte once instead of about twice. Memory doesn’t grow with output size, it needs no temp disk, and it balances load dynamically. It is more complex, though. For a few million loci in loci mode, in-memory batches are probably fine. The ordered writer pays off in region mode.
Refactor: I also suggested making the writer functions generic over Write, which the ordered writer needs. The output redundancy cleanup was a follow-up suggestion.

## Follow-up review (2026-11-06)

Second pass comparing the notes above with a separate design discussion, plus checks against the code.

### Claims checked

- **`--merge` with one reads file: confirmed by reading `src/output.rs`, not run.** With one reads file, `make_output_csv_paths` returns `--output` itself as the per-file path. `merge_output_csvs` opens it for reading, then truncates it with `File::create`, writes an empty merged file, and finally deletes it. See `docs/REVIEW.md` #6.
- **Merge alignment: confirmed.** The return value of `read_byte_record` for the second and later files is ignored, and `chrom`/`pos` are never compared. Low risk today, because every file comes from the same variant list. See `docs/REVIEW.md` #7.
- **About 120 bytes per `VariantInfo`: plausible.** The struct itself is about 88 bytes: `String` (24), `u64` (8), two `Option<String>` (48) and `VarType` padded to 8. Each chromosome name adds a small heap allocation on top. chr12 is about 133 Mb, so about 16 GB.
- **OOM before binning: consistent.** As `Vec` capacity doubles while positions are pushed, each reallocation briefly holds both the old and new buffers. Peak memory during parsing alone can reach about 25–32 GB. Fixing binning alone would not be enough; the up-front expansion has to go.

### Comparison with the alternative proposal

The alternative started from the smallest change: keep intervals, tile them into 100 kb chunks, and create a `VariantBin` per chunk so the existing `process_bin` can be reused unchanged. Peak memory is then one chunk per thread. It also considered:

- Creating positions lazily as reads arrive (a sliding window).
- htslib pileup.

The design above is the better long-term target:

- **Dense array.** The dense per-window accumulator replaces the deque and the per-position `VariantInfo`, and makes the sliding window unnecessary.
- **Bin once.** Binning once and sharing index ranges removes the copy each sample makes.
- **Ordered writer.** The work queue plus ordered writer is the more complete parallel design.

Pileup is still not recommended, because it is a rewrite of `features.rs`, not a fix.

**Suggested staging:**

1. Add windows and the dense accumulator, keeping one thread per reads file.
2. Then add the work queue and the ordered writer.

### New suggestions

1. **Walk each read's alignment once.** `Variant::count_locus_features` calls `ref_pos_to_query_pos` (`cigar.read_pos`), which scans the CIGAR once for every covered position. In region mode a read covers about 100–150 positions, so this is the hot loop. With a dense window, walk the read's aligned blocks once and update `acc[ref_pos - window_start]` directly. `BamRecordExtensions::aligned_block_pairs` is already imported in `read_parser.rs`. Check how deletions and reference skips are counted today and handle them separately.

2. **Plan for the output size.** One CSV row per base for chr12 is about 133 million rows. At a few hundred bytes per row that is tens of GB of text, the next limit after memory. Options, which can be combined:
   - Gzip/bgzip output, via `flate2`, which is already a dependency.
   - A region-mode header that drops `ref`, `alt` and `vartype`, which are always empty or `UNKNOWN` there.
   - A `--min-depth` filter.
   - Summary output with one row per BED interval or per window, like mosdepth `regions.bed`. Decide whether per-base output is actually needed before optimizing it.

3. **Check the accumulators can be added up.** A small dense accumulator only works if every output column can be built from sums and counts. `ReadFeatures` and `normalized_row` in `features.rs` (about lines 80–195) have not been checked. A median or other order statistic would need a different design. If everything is sums and counts, store each statistic as its own `u32` array (struct-of-arrays) for cache efficiency, at tens of bytes per position.

4. **Write merged output directly from the ordered writer.** Make one job cover one window across all samples. The writer then receives every sample's rows for that window together and can write the merged row straight away. That removes:
   - the per-file CSVs,
   - the separate merge pass,
   - both merge bugs.

   The cost is that a window waits for its slowest sample, and the window gate already bounds that.

5. **Readers per worker, and CRAM memory.** With a shared queue, each worker needs its own `IndexedReader` per sample. Keep those readers open for the worker's lifetime rather than reopening them per window. For CRAM, each reader also holds its own decoded reference slices, so memory grows with workers × samples. Include that when sizing the window gate.

6. **Regression test comparing region and loci mode.** For the same positions, region mode on a BED and loci mode (`--alleles none`, SNV) on a VCF should produce identical values in all shared columns. This checks the rewrite against the existing loci path, which has been checked against Python varlap. The chrM fixture can supply both inputs.

7. **Measure memory.** Record peak memory with `/usr/bin/time -v` (max resident set size) on a small BED and on chr12, before and after. Add a `size_of` check test for the accumulator struct so it can't grow by accident.