#Contig analysis

##Assemble reads into contigs with MEGAHIT.
- The `--p-num-partition` specifies the number of partitions to split the dataset into for parallel processing during assembly.
- The `--p-presets` specifies the preset mode for MEGAHIT. In this case, it's set to "meta-sensitive" for metagenomic data.
- The `--p-cpu-threads` specifies the number of CPU threads to use during assembly. 
```bash
qiime assembly assemble-megahit \
    --i-seqs "./moshpit-tutorial/cache:reads_no_host" \
    --p-presets "meta-sensitive" \
    --p-num-cpu-threads 64 \
    --p-num-partitions 4 \
    --o-contigs "./moshpit-tutorial/cache:contigs" \
    --verbose
```
Alternatively, you can also use `qiime assembly assemble-spades` to assemble contigs with SPAdes

##Contig taxonomic annotation workflow

###Classify contigs with Kraken2
- The `--p-confidence` and `--p-minimum-base-quality` are deviations from kraken's defaults.
- The database used here is the `PlusPF` database, defined [here](https://benlangmead.github.io/aws-indexes/k2).
- The abbreviations in my `output-dir` are the database (`k2pf`), and shorthand for the values I set for confidence (`c60`) and minimum base quality (`mbq20`), respectively.
```bash
qiime moshpit classify-kraken2 \
    --i-seqs "./moshpit-tutorial/cache:megahit-contigs" \
    --i-kraken2-db "./moshpit-tutorial/cache:kraken_standard" \
    --p-threads 48 \
    --p-confidence 0.6 \
    --p-minimum-base-quality 20 \
    --p-num-partitions 4 \
    --o-reports "./moshpit-tutorial/cache:kraken-reports-contigs" \
    --o-hits "./moshpit-tutorial/cache:kraken-hits-contigs" \
    --verbose
```

###Build presence/absence feature table
```bash
qiime moshpit kraken2-to-features \
  --i-reports "./moshpit-tutorial/cache:kraken_reports_contigs" \
  --o-table "./moshpit-tutorial/cache:kraken_contigs_feature_table" \
  --o-taxonomy "./moshpit-tutorial/cache:kraken_contigs_taxonomy" \
  --verbose
```

###Build taxa-bar plot
```bash
qiime taxa barplot \
    --i-table "./moshpit-tutorial/cache:kraken_contigs_feature_table" \
    --i-taxonomy "./moshpit-tutorial/cache:kraken_contigs_taxonomy "\
    --o-visualization "./moshpit-tutorial/results/kraken_contigs_taxa_barplot.qzv \
    --verbose
```

##Contig functional annotation workflow

###EggNOG search using diamond aligner
- The `--p-db-in-memory`loads the database into memory for faster processing.
```bash
qiime moshpit eggnog-diamond-search \
  --i-sequences "./moshpit-tutorial/cache:contigs" \
  --i-diamond-db "./moshpit-tutorial/cache:eggnog_diamond_full"\
  --p-num-cpus 14 \
  --p-db-in-memory \
  --o-eggnog-hits "./moshpit-tutorial/cache:contigs_diamond_hits" \
  --o-table "./moshpit-tutorial/cache:contigs_diamond_feature_table \
  --verbose
```
###Annotate orthologs against eggNOG database
```bash
qiime moshpit eggnog-annotate \
 --i-eggnog-hits "./moshpit-tutorial/cache:contigs" \
 --i-eggnog-db "./moshpit-tutorial/cache:eggnog_annot_full" \
 --o-ortholog-annotations "./moshpit-tutorial/cache:eggnog_annotated_contigs" \
 --verbose
```
