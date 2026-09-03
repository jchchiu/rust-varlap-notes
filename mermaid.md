```mermaid
flowchart TD

    classDef required stroke:#28a745,stroke-width:3px;
    classDef optional stroke:#ffc107,stroke-width:1px,stroke-dasharray: 5 5;

    subgraph Variant Handling
    direction TB

    var_file:::required@{ shape: manual-file, label: "FILE INPUT \n Variant file \n (vcf, csv, tsv) (.gz)"}
    var_file --> var_parse@{ shape: rect, label: "VARIANT PARSER" }
    var_class:::required@{ shape: manual-input, label: "USER INPUT \n Variant Class \n (Indel, Snv)"} --> var_parse
    var_parse -->|if variant passes QC| parsed_variants@{ shape: docs, label: "ParsedVariants \n Vec<VariantInfo>" }

    end

    subgraph Binning
    direction TB

    gap:::optional@{ shape: manual-input, label: "USER INPUT \n Gap (optional)" }
    parsed_variants --> bin_step@{ shape: rect, label: "BIN VARIANTS \n by chromosome + gap" }
    gap --> bin_step
    bin_step --> binned_variants@{ shape: docs, label: "BinnedVariants \n Vec<VariantBin>" }

    end

    scanning_mode@{ shape: manual-input, label: "USER INPUT \n Scanning Mode \n (Allele , Region)"} -- Region --> region_parse@{ shape: rect, label: "REGION PARSER" }
    bed_file@{ shape: manual-file, label: "FILE INPUT \n Region file \n (bed)"}
    bed_file --> region_parse

    subgraph Output
    direction TB

    csv_filename:::required@{ shape: manual-input, label: "USER INPUT \n CSV output path" } --> variant_output@{ shape: win-pane, label: "CSV output of variant statistics" }
    label:::optional@{ shape: manual-input, label: "USER INPUT \n Label(s) \n Vec<String>"}
    label --> variant_output
    merge:::optional@{ shape: manual-input, label: "USER INPUT \n Merge (optional)" }
    merge --> variant_output

    end

    subgraph Read Handling
    direction TB

    binned_variants --> bam_parse@{ shape: rect, label: "BAM PARSER"}
    reads_file:::required@{ shape: manual-file, label: "FILE INPUT(S) \n Reads files \n Vec<PathBuf> (bam, cram)"}
    fasta_file:::optional@{ shape: manual-file, label: "FILE INPUT \n Fasta file \n (fasta/fa)"}
    fasta_file -->|if provided, overrides CRAM header fasta| bam_parse
    reads_file --> bam_parse
    bam_parse --> fetch_reads@{shape: subproc, label: "Per bin: fetch region, \n iterate overlapping reads" }
    fetch_reads --> variant_conditional@{ shape: hex, label: "if read start > \n variant position" }
    variant_conditional --> remove_variant@{ shape: diamond, label: "Pop variant from queue" }
    remove_variant --> variant_conditional
    fetch_reads --> variant_iter@{ shape: hex, label: "else iterate over variants" }
    variant_iter -->|if read overlaps variant| count_features@{ shape: subproc, label: "count locus features for read" }

    end

    remove_variant -->|write variant statistics| variant_output