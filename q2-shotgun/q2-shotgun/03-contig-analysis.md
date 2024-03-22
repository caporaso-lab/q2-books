# Contig analysis (Paula)

## Assemble reads into contigs with MEGAHIT
- The `--p-num-partition` specifies the number of partitions to split the dataset into for parallel processing during assembly.
- The `--p-presets` specifies the preset mode for MEGAHIT. In this case, it's set to "meta-sensitive" for metagenomic data.
- The `--p-cpu-threads` specifies the number of CPU threads to use during assembly. 
```bash
qiime assembly assemble-megahit \
    --i-seqs "./moshpit_tutorial/cache:reads_no_host" \
    --p-presets "meta-sensitive" \
    --p-num-cpu-threads 64 \
    --p-num-partitions 4 \
    --o-contigs "./moshpit_tutorial/cache:contigs" \
    --verbose
```
Alternatively, you can also use `qiime assembly assemble-spades` to assemble contigs with SPAdes.
## Contig QC with QUAST
```bash
qiime assembly evaluate-contigs \
    --i-contigs "./moshpit_tutorial/cache:contigs" \
    --p-threads 128 \
    --p-memory-efficient \
    --o-visualization ""./moshpit_tutorial/results/contigs.qzv" \
    --verbose
```
## Contig taxonomic annotation workflow

### Classify contigs with Kraken2
- The `--p-confidence` and `--p-minimum-base-quality` are deviations from kraken's defaults.
- The database used here is the `PlusPF` database, defined [here](https://benlangmead.github.io/aws-indexes/k2).
- The abbreviations in my `output-dir` are the database (`k2pf`), and shorthand for the values I set for confidence (`c60`) and minimum base quality (`mbq20`), respectively.
```bash
qiime moshpit classify-kraken2 \
    --i-seqs "./moshpit_tutorial/cache:megahit-contigs" \
    --i-kraken2-db "./moshpit_tutorial/cache:kraken_standard" \
    --p-threads 48 \
    --p-confidence 0.6 \
    --p-minimum-base-quality 20 \
    --p-num-partitions 4 \
    --o-reports "./moshpit_tutorial/cache:kraken_reports_contigs" \
    --o-hits "./moshpit_tutorial/cache:kraken_hits_contigs" \
    --verbose
```
Alternatively, you can also use `qiime moshpit classify-kaiju` to classify your contigs with Kaiju.

### Build presence/absence feature table
```bash
qiime moshpit kraken2-to-features \
  --i-reports "./moshpit_tutorial/cache:kraken_reports_contigs" \
  --o-table "./moshpit_tutorial/cache:kraken_feature_table_contigs" \
  --o-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_contigs" \
  --verbose
```

### Build taxa-bar plot
```bash
qiime taxa barplot \
    --i-table "./moshpit_tutorial/cache:kraken_feature_table_contigs" \
    --i-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_contigs "\
    --o-visualization "./moshpit_tutorial/results/kraken_taxa_barplot_contigs.qzv \
    --verbose
```

## Contig functional annotation workflow

### EggNOG search using diamond aligner
- The `--p-db-in-memory`loads the database into memory for faster processing.
```bash
qiime moshpit eggnog-diamond-search \
  --i-sequences "./moshpit_tutorial/cache:contigs" \
  --i-diamond-db "./moshpit_tutorial/cache:eggnog_diamond_full"\
  --p-num-cpus 14 \
  --p-db-in-memory \
  --o-eggnog-hits "./moshpit_tutorial/cache:diamond_hits_contigs" \
  --o-table "./moshpit_tutorial/cache:diamond_feature_table_contigs \
  --verbose
```
### Gene diversity
Generate a unweighted beta diversity matrix, ie., Jaccard
```bash
qiime diversity beta \
  --i-table "./moshpit_tutorial/cache:diamond_feature_table_contigs" \
  --p-metric jaccard \
  --o-distance-matrix "./moshpit_tutorial/cache:jaccard_distance_matrix_contigs"
```
Generate a PCoA from Jaccard matrix output
```bash
qiime diversity pcoa \
  --i-distance-matrix "./moshpit_tutorial/cache:jaccard_distance_matrix_contigs" \
  --o-pcoa "./moshpit_tutorial/cache:jaccard_distance_matrix_pcoa_contigs"
```
Visualize PCoA using emperor
```bash
qiime emperor plot \
  --i-pcoa  "./moshpit_tutorial/cache:jaccard_distance_matrix_pcoa_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --o-visualization "./moshpit_tutorial/results/jaccard_distance_matrix_pcoa_contigs.qzv"
```

### Annotate orthologs against eggNOG database
```bash
qiime moshpit eggnog-annotate \
 --i-eggnog-hits "./moshpit_tutorial/cache:contigs" \
 --i-eggnog-db "./moshpit_tutorial/cache:eggnog_annot_full" \
 --o-ortholog-annotations "./moshpit_tutorial/cache:eggnog_annotated_contigs" \
 --verbose
```

