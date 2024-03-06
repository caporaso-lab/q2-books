# MAG analysis

## Bin contigs into MAGs
Do we talk about `--parallel-config`?? 

### Contigs indexing
```bash
qiime assembly index-contigs \
    --i-contigs "./moshpit_tutorial/cache:contigs" \
    --p-seed 100 \
    --p-threads 64 \
    --p-verbose \
    --p-num-partitions 4 \
    --o-index "./moshpit_tutorial/cache:contigs_index" \
    --verbose
```    
### Read mapping to contigs
```bash
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
```bash
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

## MAGs dereplication (optional)
```bash
qiime sourmash compute \
    --i-sequence-file "./moshpit_tutorial/cache:mags" \
    --p-ksizes 35 \
    --p-scaled 10 \
    --o-min-hash-signature "./moshpit_tutorial/cache:mags_minhash" \
    --verbose
```
```bash
qiime sourmash compare \
    --i-min-hash-signature "./moshpit_tutorial/cache:mags_minhash" \
    --p-ksize 35 \
    --o-compare-output "./moshpit_tutorial/cache:mags_dist_matrix" \
    --verbose
```
```bash
qiime moshpit dereplicate-mags \
    --i-mags "./moshpit_tutorial/cache:mags" \
    --i-distance-matrix "./moshpit_tutorial/cache:mags_dist_matrix" \
    --p-threshold 0.99 \
    --o-dereplicated-mags "./moshpit_tutorial/cache:mags_derep" \
    --o-feature-table "./moshpit_tutorial/cache:mags_ft" \
    --verbose
```

## MAGs QC with BUSCO
```bash
qiime moshpit evaluate-busco \
    --i-bins "./moshpit_tutorial/cache:mags" \
    --p-lineage-dataset bacteria_odb10 \
    --p-cpu 196 \
    --o-visualization "/cluster/work/bokulich/mziemski/_data/manuscript/nf-fmt-samples/results/mags.qzv" \
    --verbose
```
In this tutorial we perfom taxonomic and functional annotation on dereplicated MAGs
## MAGs taxonomic annotation workflow

### MAGs classify with Kraken2
```bash
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
```bash
qiime moshpit kraken2-to-mag-features \
 --i-reports "./moshpit_tutorial/cache:kraken_reports_derep_mags" \
 --i-hits "./moshpit_tutorial/cache:kraken_hits_derep_mags" \
 --o-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_derep_mags" \
 --verbose
```
### Compare taxonomy contigs vs MAGs with RESCRIPt
A tutorial demonstrating some of the basic functionality of RESCRIPt is available [here](https://library.qiime2.org/plugins/rescript/27/).
```bash
qiime rescript evaluate-taxonomy \
 --i-taxonomies "/cluster/work/bokulich/mziemski/_data/manuscript/cache-fmt-full:kraken_contigs_taxonomy" "/cluster/work/bokulich/mziemski/_data/manuscript/cache-fmt-full:kraken_taxonomy_mags" \
 --o-taxonomy-stats "/cluster/work/bokulich/mziemski/_data/manuscript/nf-fmt-samples/results/taxonomy_stats.qzv" \
 --verbose
```
## MAGs functional annotation workflow

### EggNOG search using diamond aligner
```bash
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
```bash
qiime moshpit eggnog-annotate \
 --i-eggnog-hits "./moshpit_tutorial/cache:derep_mags" \
 --i-eggnog-db "./moshpit_tutorial/cache:eggnog_annot_full" \
 --o-ortholog-annotations "./moshpit_tutorial/cache:eggnog_annotated_derep_mags" \
 --verbose
```
