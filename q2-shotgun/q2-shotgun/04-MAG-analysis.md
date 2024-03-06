# MAG analysis

## Bin contigs into MAGs

### Contigs indexing
TODO
### Read mapping to contigs
TODO
### Binning
TODO

## MAGs dereplication (optional)
TODO

## MAGs QC with BUSCO
TODO

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
A tutorial demonstrating some of the basic functionality of RESCRIPt is available here.
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
