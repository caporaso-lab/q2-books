# MAG analysis
```{warning}
Metagenomics analysis with QIIME 2 is in alpha release.
This means that results you generate should be considered preliminary, and NOT PUBLICATION QUALITY.
Additionally, interfaces are subject to change, and those changes may be backward incompatible (meaning that a command or file that works in one version of the QIIME 2 Shotgun Metagenomics distribution may not work in the next version of that distribution).
```

This section of the tutorial focuses on analyzing metagenome-assembled genomes (MAGs), which are reconstructed genomes derived from metagenomic data.

## Bin contigs into MAGs
Let's start binning our contigs into MAGs using various tools and methodologies!

### Read mapping
We first need to index the contigs obtained in the assembly step and map the original reads to those contigs using that index. This read mapping can then be used by the contig binner to figure out which contigs originated from the same genome and put those together.

```shell
qiime assembly index-contigs \
    --i-contigs "./moshpit_tutorial/cache:contigs" \
    --p-seed 100 \
    --p-threads 64 \
    --p-verbose \
    --p-num-partitions 4 \
    --o-index "./moshpit_tutorial/cache:contigs_index" \
    --verbose
```
```shell
qiime assembly map-reads-to-contigs \
    --i-indexed-contigs "./moshpit_tutorial/cache:contigs_index" \
    --i-reads "./moshpit_tutorial/cache:reads_no_host" \
    --p-seed 100 \
    --p-threads 64 \
    --p-num-partitions 4 \
    --o-alignment-map "./moshpit_tutorial/cache:reads_to_contigs" \
    --verbose
```
### Binning
Finally, we are ready to perform contig binning. This process involves categorizing contigs into distinct bins or groups based on their likely origin from different microbial species or strains within a mixed community. Here, we will use the [MetaBAT 2](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6662567/) tool, which uses tetranucleotide frequency together with abundance (coverage) information to assign contigs to individual bins.

```shell
qiime moshpit bin-contigs-metabat \
    --i-contigs "./moshpit_tutorial/cache:contigs" \
    --i-alignment-maps "./moshpit_tutorial/cache:reads_to_contigs" \
    --p-seed 100 \
    --p-num-threads 128 \
    --p-verbose \
    --o-mags "./moshpit_tutorial/cache:mags" \
    --o-contig-map "./moshpit_tutorial/cache:contig_map" \
    --o-unbinned-contigs "./moshpit_tutorial/cache:unbinned_contigs" \
    --verbose
```
This tep generated a couple artifacts:

- `mags.qza`: these are our actual MAGS, per sample.
- `contig-map.qza`: this is a mapping between MAG IDs and IDs of contigs which belong to a given MAG.
- `unbinned-contigs.qza`: these are all the contigs that could not be assign to any particular MAG.
From now on, we will focus on the mags.qza.

## MAGs dereplication (optional)
Dereplication involves removing duplicate or nearly identical MAGs to reduce redundancy and improve downstream analyses. To dereplicate our MAGs, we will:

1. compute hash sketches of every genome using [sourmash](https://sourmash.readthedocs.io/en/latest/) - you can think of those sketches as tiny representations of our genomes (sourmash compresses a lot of information into much smaller space).
2. compare all of those sketches (genomes) to one another to generate a matrix of pairwise distances between our MAGs
3. dereplicate the genomes using the distance matrix and a fixed similarity threshold: the last action will simply choose the longest genome from all of the genomes belonging to the same cluster, given a similarity threshold.

```shell
qiime sourmash compute \
    --i-sequence-file "./moshpit_tutorial/cache:mags" \
    --p-ksizes 35 \
    --p-scaled 10 \
    --o-min-hash-signature "./moshpit_tutorial/cache:mags_minhash" \
    --verbose
```
```shell
qiime sourmash compare \
    --i-min-hash-signature "./moshpit_tutorial/cache:mags_minhash" \
    --p-ksize 35 \
    --o-compare-output "./moshpit_tutorial/cache:mags_dist_matrix" \
    --verbose
```
```shell
qiime moshpit dereplicate-mags \
    --i-mags "./moshpit_tutorial/cache:mags" \
    --i-distance-matrix "./moshpit_tutorial/cache:mags_dist_matrix" \
    --p-threshold 0.99 \
    --o-dereplicated-mags "./moshpit_tutorial/cache:mags_derep" \
    --o-feature-table "./moshpit_tutorial/cache:mags_ft" \
    --verbose
```

## MAGs QC with BUSCO
BUSCO is used here to assess the completeness and quality of MAGs by searching for single-copy orthologous genes within the genomes.
```shell
qiime moshpit evaluate-busco \
    --i-bins "./moshpit_tutorial/cache:mags" \
    --p-lineage-dataset bacteria_odb10 \
    --p-cpu 196 \
    --o-visualization "./moshpit_tutorial/results/mags.qzv" \
    --verbose
```
## MAGs taxonomic annotation workflow
This workflow focuses on annotating MAGs with taxonomic information using Kraken2, a tool for taxonomic classification. In this tutorial we perfom taxonomic and functional annotation on dereplicated MAGs.
### MAGs classify with Kraken2
MAGs are classified taxonomically using Kraken2, with parameters set for confidence threshold and minimum base quality.
```shell
qiime moshpit classify-kraken2 \
    --i-seqs "./moshpit_tutorial/cache:derep_mags" \
    --i-kraken2-db "./moshpit_tutorial/cache:kraken_standard" \
    --p-threads 48 \
    --p-confidence 0.6 \
    --p-minimum-base-quality 20 \
    --p-num-partitions 4 \
    --o-reports "./moshpit_tutorial/cache:kraken_reports_derep_mags" \
    --o-hits "./moshpit_tutorial/cache:kraken_hit_derep_mags" \
    --verbose
```
Alternatively, you can also use `qiime moshpit classify-kaiju` to classify your contigs with Kaiju.

### Build taxonomy table
A taxonomy table is constructed based on the classification results, providing taxonomic assignments for each MAG.
```shell
qiime moshpit kraken2-to-mag-features \
 --i-reports "./moshpit_tutorial/cache:kraken_reports_derep_mags" \
 --i-hits "./moshpit_tutorial/cache:kraken_hits_derep_mags" \
 --o-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_derep_mags" \
 --verbose
```

## MAGs functional annotation workflow
This workflow involves annotating MAGs with functional information, particularly identifying functional elements like genes and their annotations.
### EggNOG search using diamond aligner
MAGs are searched against the EggNOG database using the Diamond aligner to identify functional annotations.
```shell
qiime moshpit eggnog-diamond-search \
  --i-sequences "./moshpit_tutorial/cache:derep_mags" \
  --i-diamond-db "./moshpit_tutorial/cache:eggnog_diamond_full"\
  --p-num-cpus 14 \
  --p-db-in-memory \
  --o-eggnog-hits "./moshpit_tutorial/cache:diamond_hits_derep_mags" \
  --o-table "./moshpit_tutorial/cache:diamond_feature_table_derep_mags \
  --verbose
```
### Annotate orthologs against eggNOG database
Orthologs in MAGs are annotated against the EggNOG database, providing functional insights into the genes and gene products present in the MAGs.
```shell
qiime moshpit eggnog-annotate \
 --i-eggnog-hits "./moshpit_tutorial/cache:derep_mags" \
 --i-eggnog-db "./moshpit_tutorial/cache:eggnog_annot_full" \
 --o-ortholog-annotations "./moshpit_tutorial/cache:eggnog_annotated_derep_mags" \
 --verbose
```
