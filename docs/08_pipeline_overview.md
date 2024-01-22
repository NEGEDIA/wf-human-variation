The workflow is composed of 6 distinct subworkflows, each enabled by a command line option:

* [SNP calling](#3-small-variant-calling-with-clair3): `--snp`
* [SV calling](#4-structural-variant-sv-calling-with-sniffles2): `--sv`
* [Analysis of modified bases](#5-modified-base-calling-with-modkit): `--mod`
* [CNV calling](#6-copy-number-variants-cnv-calling-with-qdnaseq): `--cnv`
* [STR genotyping](#7-short-tandem-repeat-str-genotyping-with-straglr): `--str`

Subworkflows where the relevant option is omitted will not be run.

### 1. Input and data preparation

The workflow relies on three primary input files:
1. A reference genome in [FASTA format](https://www.ncbi.nlm.nih.gov/genbank/fastaformat/)
2. Sequencing data for the sample in the form of a single [BAM or CRAM file](https://samtools.github.io/hts-specs/SAMv1.pdf), either aligned or unaligned.

The input BAM file can be generated using the [wf-basecalling](https://github.com/epi2me-labs/wf-basecalling/) workflow, which is up to date with the current dorado releases and models.

### 2. Data QC and pre-processing
The workflow starts by performing multiple checks of the input BAM file, as well as computing:
1. depth of sequencing with [mosdepth](https://github.com/brentp/mosdepth);
2. read alignment statistics with [fastcat](https://github.com/epi2me-labs/fastcat).

After computing the coverage, the workflow will check that the input BAM file has a depth greater than `--bam_min_coverage`.
In case the user specify `--bam_min_coverage 0`, the check will be skipped and the workflow will proceed directly to the downstream analyses.
Some components work better withing certain ranges of coverage, and the user might achieve better results by providing a target coverage to downsample to. The user can set `--downsample_coverage true` to enable the downsampling of the reads, and `--downsample_coverage_target {{ X }}` to specify the target coverage (default: 60x).

### 3. Small variant calling with Clair3

The workflow implements a deconstructed version of [Clair3](https://github.com/HKU-BAL/Clair3) (v1.0.4) to call germline variants. The appropriate model can be provided with the `--basecaller_cfg` option. To decide on the appropriate model you can check out the Dorado documentation for a list of available basecalling models.
This workflow takes advantage of the parallel nature of Nextflow, providing optimal efficiency in high-performance, distributed systems. The workflow will automatically call small variants (SNPs and indels), collect statistics, annotate them with [SnpEff](https://pcingola.github.io/SnpEff/) (and additionally for SNPs, ClinVar details), and create a report summarising the findings.

If desired, the workflow can perform phasing of structural variants by using the `--phased` option. This will lead the workflow to use [longphase](https://github.com/twolinin/longphase) to perform phasing of the variants, with the option to use [whatshap](https://whatshap.readthedocs.io/) instead by setting `--use_longphase false`. The phasing will also generate a GFF file with the annotation of the phase blocks, facilitating the detection of these within genome visualizers.

### 4. Structural variant (SV) calling with Sniffles2

The workflow allows for calling of SVs using long-read sequencing data with [Sniffles2](https://github.com/fritzsedlazeck/Sniffles).
The workflow will perform SV calling, filtering and generation of a report.
Optionally, the workflow can also evaluate calls on HG002 against a truth set (provided the input data was aligned to HG19).
The SV workflow takes an optional `--tr_bed` option to specify tandem repeats in the reference sequence --- see the [sniffles](https://github.com/fritzsedlazeck/Sniffles) documentation for more information.
SVs can be phased using `--phased`. However, this will cause the workflow to run SNP analysis, as SV phasing relies on the haplotagged reads generated in this stage.

### 5. Modified base calling with modkit

Modified base calling can be performed by specifying `--mod`. The workflow will call modified bases using [modkit](https://github.com/nanoporetech/modkit). 
The workflow will automatically check whether the files contain the appropriate `MM`/`ML` tags, required for running [modkit pileup](https://nanoporetech.github.io/modkit/intro_bedmethyl.html). If the tags are not found, the workflow will not run the individual analysis, but will still run the other subworkflows requested by the user.
The default behaviour of the workflow is to run modkit with the `--cpg --combine-strands` options set. It is possible to report strand-aware modifications by providing `--force_strand`, which will trigger modkit to run in default mode. The resulting bedMethyl will include modifications for each site on each strand separately.
The modkit run can be fully customized by providing `--modkit_args`. This will override any preset, and allow full control over the run of modkit.
Haplotype-resolved aggregated counts of modified bases can be obtained with the `--phased` option. This will generate three distinct BEDMethyl files with the naming pattern `{{ alias }}_{{ haplotype }}.wf_mods.bedmethyl.gz`, where `haplotype` can be `1`, `2` or `ungrouped`.

### 6. Copy number variants (CNV) calling with QDNASeq

CNV calling is performed using [QDNAseq](https://github.com/ccagc/QDNAseq). This workflow is compatible with genome builds hg19/GRCh37 or hg38/GRCh38.
In addition to the VCF of CNV calls, the workflow emits QDNAseq-generated plots and BED files of both raw read counts per bin and corrected, normalised, and smoothed read counts per bin.

### 7. Short tandem repeat (STR) genotyping with Straglr

STR genotyping is performed using a fork of [straglr](https://github.com/philres/straglr). This workflow is compatible with genome build hg38/GRCh38.
The STR workflow takes a required `--sex` option which is `male` or `female`. If `--sex` is not specified, the workflow will default to `female`. Please be aware that incorrect sex assignment will result in the wrong number of calls for all repeats on chrX.
In addition to a gzipped VCF file containing STRs found in the dataset, the workflow emits a TSV straglr output containing reads spanning STRs, and a haplotagged BAM. 

### 8. Phasing variants
The workflow can perform joint physical phasing with `longphase` of SNP, Indels and SVs by setting the `--phased --snp --sv` options.
The behaviour of the phasing is summarised in the below table:

|  |  |  |  | phased SNP VCF | phased SV VCF | Joint SV+SNP phased VCF | Phased bedMethyl |
|---------|--------|---------|------------|--------|--------|--------|--------|
| `--snp` | `--sv` | `--mod` | `--phased` |  |  | &check; | &check; |
| `--snp` | `--sv` |  | `--phased` |  |  | &check; |  |
| `--snp` |  |  | `--phased` | &check; |  |  |
|  | `--sv` |  | `--phased` |  | &check; |  |  |  |
|  |  | `--mod` | `--phased` |  |  |  | &check; |

In some circumstances, users may wish to keep the separate VCF files before joint phasing. This can be done with `--output_separate_phased`.

### 9. Variant annotation
Annotation will be performed automatically by the SNP and SV subworkflows, and can be disabled by the user with `--annotation false`. The workflow will annotate the variants using [SnpEff](https://pcingola.github.io/SnpEff/), and currently only support the human hg19 and hg38 genomes. Additionally, the workflow will add the [ClinVar](https://www.ncbi.nlm.nih.gov/clinvar/) annotations for the SNP variants.

Running the workflow on non-human samples will require this option to be disabled.