# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

rust-varlap is a Rust rewrite of the Python tool [varlap](https://github.com/bjpop/varlap). It computes per-locus read QC statistics for variants or regions from BAM/CRAM files and writes them as CSV. Matching the Python varlap's output is a primary correctness goal.

## Commands

- Build (release, used for real data / timing): `cargo build -r` → `./target/release/rust-varlap`
- Tests (unit tests live in `src/variant_parser.rs`, `src/binning.rs` and `src/region_parser.rs`; `tests/integration_test.rs` is currently empty): `cargo test`
- Single test: `cargo test <test_name>` (e.g. `cargo test binning::tests::`)
- Lint: `cargo clippy`
- Build needs htslib system libs (via rust-htslib): zlib, bz2, lzma, curl, ssl, deflate, cmake, pkg-config. See `.github/workflows/release.yml`.
- Release: pushing a `v*` tag triggers `.github/workflows/release.yml`, which builds the Linux binary and an Apptainer `.sif`. The version string is in both `Cargo.toml` and the `#[command(version)]` in `src/cli.rs`, so keep the two in sync.

### Running

The CLI has two subcommands (`src/cli.rs`). They share `CommonArgs`: `-r/--reads` (one or more), `-o/--output`, `-f/--fasta` (CRAM), `--sample`, `--label`, `--gap`, `--merge` and `-t/--threads`.

    rust-varlap loci   -v variants.vcf[.gz|csv|tsv] -c snv|indel [--alleles strict|partial|none] -r a.bam -o out.csv
    rust-varlap region -R regions.bed -r a.bam -o out.csv

The README's Usage section still shows the old flat-flag CLI, without subcommands.

### Parity check against Python varlap

`test_data/` has a small chrM stress fixture: a BAM plus index, a VCF, and the Python varlap output `python.chrM_heavy_stress.snv.csv`. Compare with:

    ./target/release/rust-varlap loci -v test_data/chrM_heavy_stress.vcf -r test_data/chrM_heavy_stress.sorted.bam -c snv -o /tmp/out.csv
    tests/compare_outputs.sh /tmp/out.csv test_data/python.chrM_heavy_stress.snv.csv /tmp/diff.txt

`compare_outputs.sh` diffs the two files field by field and skips the header row, because the header labels intentionally differ from Python (`ref_avg_nm` rather than `ref avg nm`). `test_data/notes.md` covers larger real-data comparisons, including a known base-quality difference caused by lossy 1000 Genomes CRAMs.

## Architecture

The pipeline lives in `src/main.rs::run`, which is one match arm per subcommand. Each step:

1. **Parse** → `ParsedVariants` (a `Vec<VariantInfo>` with chrom, 1-based `pos`, optional `refr`/`alt`, and `VarType`).
   - `variant_parser.rs`: VCF/CSV/TSV, which may be gzipped (detected by magic bytes in `reader.rs`). It also handles allele normalization, varclass filtering and the `AlleleMode` handling of missing REF/ALT, and it enforces sorted input (`AppError::UnsortedVariants`).
   - `region_parser.rs`: BED. It expands each region into one `VariantInfo` per base, with no alleles and `VarType::Unknown`. Region mode then runs the SNV feature path. This expansion is the known memory problem on the `add-region-mode` branch, and batching is the likely fix.
2. **Bin** (`binning.rs::bin`): groups variants into `VariantBin`s by chromosome and splits wherever consecutive variants are more than `gap` apart (default 100 kb). Each `Variant<'a>` borrows its `VariantInfo` and owns a fresh `LocusFeatures` (Snv or Indel, chosen from the varclass). Binning runs again for each reads file, so the counters are never shared.
3. **Read scan** (`read_parser.rs`): each bin does one `IndexedReader::fetch` over its span. The algorithm iterates over **reads, not pileups**. Before processing a read, it pops and writes every variant at the front of the `VecDeque` whose position is before the read's start; that variant can't gain more coverage, so its CSV row is written immediately. The read then counts features for each queued variant it overlaps. `skip_read_check` mirrors pysam pileup defaults: it skips orphans, unmapped, secondary, QC-fail and duplicate reads. Ten consecutive read errors raise `TruncatedReads`.
4. **Features** (`features.rs`): `LocusFeatures::{Snv, Indel}` wrap a common `ReadFeatures` (MAPQ, NM, base quality, alignment length and so on) along with allele counts (`AlleleCountsSnv`), or indel event matching (`LocusFeaturesIndel`). `Variant::count_locus_features` in `variant.rs` maps ref→query position with the cached CIGAR.
5. **Output** (`output.rs`): one CSV per reads file, with column labels prefixed by the reads label (defaulting to the filename). `--merge` joins the per-file CSVs at the end.

Reads files run in parallel through a global rayon pool (`--threads`). Each file gets its own reader, bins and output CSV.

## Conventions

- Errors: domain errors are `AppError` variants (`src/errors.rs`, thiserror), and each has an `exit_code()` and a custom printer in `print_error`. Everything else uses `anyhow` with `.context(...)`. Add new failure kinds as `AppError` variants and give each one an exit code and a printer arm.
- Logging goes through `tracing` (`info!`/`debug!`/`warn!`). Skipped input rows are logged with `warn!` and counted, and they don't raise errors.
- Input assumptions: the variants/BED and reads files are coordinate-sorted, and BAM/CRAM/FASTA (and gzipped VCF) files have indexes.
