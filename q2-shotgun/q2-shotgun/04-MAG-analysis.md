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

### Build presence/absence feature table
```bash
qiime moshpit kraken2-to-mag-features \
  --i-reports "./moshpit_tutorial/cache:kraken_reports_derep_mags" \
  --o-table "./moshpit_tutorial/cache:kraken_feature_table_derep_mags" \
  --o-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_derep_mags" \
  --verbose
```

### Build taxa-bar plot
```bash
qiime taxa barplot \
    --i-table "./moshpit_tutorial/cache:kraken_feature_table_derep_mags" \
    --i-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_derep_mags "\
    --o-visualization "./moshpit_tutorial/results/kraken_taxa_barplot_derep_mags.qzv \
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
  --o-eggnog-hits "./moshpit_tutorial/cache:derep_mags_diamond_hits" \
  --o-table "./moshpit_tutorial/cache:derep_mags_diamond_feature_table \
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
