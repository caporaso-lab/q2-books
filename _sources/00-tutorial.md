# Taxonomic annotation of reads and assembled contigs

```{warning}
Metagenomics analysis with QIIME 2 is in alpha release.
This means that results you generate should be considered preliminary, and NOT PUBLICATION QUALITY.
Additionally, interfaces are subject to change, and those changes may be backward incompatible (meaning that a command or file that works in one version of the QIIME 2 Shotgun Metagenomics distribution may not work in the next version of that distribution).
```

```{warning}
Filtering of human and other host-associated reads from metagenomics data is an active area of research,
and for human hosts in particular updated approaches need to be developed and validated that filter based on
the [human pangenome](https://humanpangenome.org/), rather than a single human genome sequence. We don't have
a recommendation on a tool for this right now, but some are in development. Recent publications ([1](https://genome.cshlp.org/content/29/6/954.long), [2](https://doi.org/10.1038/s41564-023-01381-3))
suggest that **personally identifying information can persist in human-derived metagenome data,
even when following current best practices for host read removal**. QIIME 2 has not solved this problem.

All of that said, you can use [`qiime quality-control filter-reads`](https://docs.qiime2.org/2023.9/plugins/available/quality-control/filter-reads/)
to filter reads that hit the host genome based on alignment to that genome. [`qiime quality-control bowtie2-build`](https://docs.qiime2.org/2023.9/plugins/available/quality-control/bowtie2-build/)
can help you build a bowtie2 database for use with `filter-reads`.

We also recommend filtering again following taxonomy assignment using [`qiime taxa filter-seqs`](https://docs.qiime2.org/2023.9/plugins/available/taxa/filter-seqs/)
to only include features that are assigned to a bacterial phylum, for example (see [here](https://docs.qiime2.org/2023.9/tutorials/filtering/#taxonomy-based-filtering-of-tables-and-sequences)).
Note that this is not perfect due to the issues pointed out [in the first publication referenced above](https://genome.cshlp.org/content/29/6/954.long).

In summary, you can do host read filtering with QIIME 2, but the current best practices
in general (not only in QIIME 2) are known to be insufficient. Developing and validating
new best practices is ongoing work.
```

## Importing data

Using code from [this repo](https://github.com/gregcaporaso/fq-manifestor), create a manifest.
```python
from fq_manifestor import fq_manifestor
fq_manifestor(
    "fastq-dir/",
    "manifest.tsv",
    f_read_pattern='paired_1',
    r_read_pattern='paired_2')
```

```shell
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path manifest.tsv \
  --output-path demux.qza \
  --input-format PairedEndFastqManifestPhred33V2
```

```shell
qiime demux summarize \
  --i-data demux.qza \
  --o-visualization demux.qzv
```

Generate a table of the sequence counts per sample so we can use that as metadata later (e.g., we should always color points in PCoA by read count, to get a feel for whether that is impacting our results).
```shell
qiime demux tabulate-read-counts \
  --i-sequences demux.qza \
  --o-counts demultiplexed-read-counts.qza
```

## Read taxonomic annotation workflow

### Classify reads with kraken2
- The `--p-confidence` and `--p-minimum-base-quality` are deviations from kraken's defaults.
- The database used here is the `PlusPF` database, defined [here](https://benlangmead.github.io/aws-indexes/k2).
- The abbreviations in my `output-dir` are the database (`k2pf`), and shorthand for the values I set for confidence (`c60`) and minimum base quality (`mbq20`), respectively.

```shell
qiime moshpit classify-kraken2 \
     --i-seqs demux.qza \
     --i-kraken2-db /projects/microbiome/biological-reference-data/2023.06.05-k2-plus-pf-kraken-db.qza \
     --p-threads 40 \
     --p-confidence 0.6 \
     --p-minimum-base-quality 20 \
     --output-dir kraken-k2pf-c60-mbq20-reads \
     --p-report-minimizer-data
```

### Estimate abundances with bracken
- As far as I know, read length should be the single-end read length for the data (look in the `demux.qzv` generated above, if you don't know this offhand).
```shell
qiime moshpit estimate-bracken \
    --i-bracken-db /projects/microbiome/biological-reference-data/2023.06.05-k2-plus-pf-bracken-db.qza \
    --p-read-len 150 \
    --i-kraken-reports kraken-k2pf-c60-mbq20-reads/reports.qza \
    --o-reports kraken-k2pf-c60-mbq20-reads/bracken-reports.qza \
    --o-taxonomy kraken-k2pf-c60-mbq20-reads/taxonomy-bracken.qza \
    --o-table kraken-k2pf-c60-mbq20-reads/table-bracken.qza
```

## Contig taxonomic annotation workflow

### Assemble reads with megahit
```shell
qiime assembly assemble-megahit \
    --i-seqs demux.qza \
    --p-num-cpu-threads 40 \
    --o-contigs megahit-contigs.qza
```

### Classify contigs with kraken2
- The `--p-confidence` value is a deviation from kraken's default.
- The database used here is the `PlusPF` database, defined [here](https://benlangmead.github.io/aws-indexes/k2).
- The abbreviations in my `output-dir` are the database (`k2pf`), and shorthand for the value I set for confidence (`c60`).
- Decreasing the confidence should increase sensitivity at the expense of specificity.
```shell
qiime moshpit classify-kraken2 \
    --i-seqs megahit/megahit-contigs.qza \
    --i-kraken2-db /projects/microbiome/biological-reference-data/2023.06.05-k2-plus-pf-kraken-db.qza \
    --p-threads 40 \
    --p-confidence 0.6 \
    --output-dir kraken-k2pf-c60-contigs/ \
    --p-report-minimizer-data \
    --verbose
```

### Build presence/absence feature table
- The abbreviations in my output file names are shorthand for the value I set for coverage threshold (`ct01`).
- Decreasing the coverage threshold should increase sensitivity at the expense of specificity.
```shell
qiime moshpit kraken2-to-features \
    --i-reports kraken-k2pf-c60-contigs/reports.qza \
    --o-table kraken-k2pf-c60-contigs/pa-table-ct01.qza \
    --o-taxonomy kraken-k2pf-c60-contigs/taxonomy-ct01.qza \
    --p-coverage-threshold 0.01
```

