# Metagenomics with QIIME 2 - Leiden 2024

## Setup

Before we dive into the tutorial, let's set up the required directory structre and make sure we have all the required software installed.

### Directory structure
```shell
<your home directory>
└ workshop
```

## Metadata
```shell
wget -O sample-metadata.tsv https://polybox.ethz.ch/index.php/s/79s2cQry8Ll0FGq/download
```

```shell
qiime metadata tabulate \
  --m-input-file sample-metadata.tsv \
  --o-visualization sample-metadata.qzv
```

```{note}
https://workshop-server.qiime2.org/<your user namey>/
```
## Read-based analysis
TODO : add hidden kraken2+bracken!

### Obtaining your Feature Table and Taxonomy Table
```shell
wget -O bracken-feature-table.qza https://polybox.ethz.ch/index.php/s/4Y1IGtZHTzo1KTi/download
```
```shell
wget -O bracken-taxonomy.qza https://polybox.ethz.ch/index.php/s/haWDZzLcJsuiI9b/download
```
### Filtering Feature Table
```shell
qiime feature-table filter-samples \
  --i-table bracken-feature-table.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table bracken-autofmt-feature-table.qza
```

### Taxa-bar Creation
```shell
qiime taxa barplot \
  --i-table bracken-autofmt-feature-table.qza \
  --i-taxonomy bracken-taxonomy.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization taxa-bar-plot-autofmt-reads.qzv
```
### Differential abundance
#### Feature Table preparation
```shell
qiime feature-table filter-samples \
  --i-table bracken-autofmt-feature-table.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where "[categorical-time-relative-to-fmt]='peri'" \
  --o-filtered-table peri-fmt-table.qza
```

```shell
qiime feature-table summarize \
  --i-table peri-fmt-table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization peri-fmt-table.qzv
```

```shell

echo SampleID > samples-to-keep.tsv
echo SRR14092317 >> samples-to-keep.tsv

```
```shell
qiime feature-table filter-samples \
  --i-table peri-fmt-table.qza \
  --m-metadata-file samples-to-keep.tsv \
  --p-exclude-ids \
  --o-filtered-table id-filtered-peri-fmt-table.qza
```

```shell
qiime feature-table summarize \
  --i-table id-filtered-peri-fmt-table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization id-filtered-peri-fmt-table.qzv
```

```shell
qiime taxa collapse \
--i-table id-filtered-peri-fmt-table.qza \
--i-taxonomy bracken-taxonomy.qza \
--p-level 8 \
--o-collapsed-table collapsed-8-id-filtered-peri-fmt-table.qza 
```
#### ANCOM-BC
```shell
 qiime composition ancombc \
  --i-table collapsed-8-id-filtered-peri-fmt-table.qza  \
  --m-metadata-file sample-metadata.tsv \
  --p-formula autoFmtGroup \
  --o-differentials differentials-peri-autofmt.qza
```

```shell
qiime composition da-barplot \
  --i-data differentials-peri-autofmt.qza \
  --p-significance-threshold 0.05 \
  --p-level-delimiter ";" \
  --o-visualization differentials-peri-autofmt.qzv
```
## Contig-based analysis
TODO : add hidden kraken2

### QUAST QC
```shell
wget -O quast-qc.qzv https://polybox.ethz.ch/index.php/s/XyZfYkDEHh1nHZq/download
```
### Obtaining your Feature Table and Taxonomy Table
```shell
wget -O kraken2-presence-absence-contigs.qza https://polybox.ethz.ch/index.php/s/OYL590hv7eZJPWS/download
```
```shell
wget -O kraken2-taxonomy-contigs.qza https://polybox.ethz.ch/index.php/s/Wk0nsgQfEjgdabc/download
```
### Filtering Feature Table
```shell
qiime feature-table filter-samples \
  --i-table kraken2-presence-absence-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table kraken2-autofmt-presence-absence-contigs.qza
```
### Alpha Diversity
#### Observed Features 
```shell
qiime diversity alpha \
    --i-table kraken2-autofmt-presence-absence-contigs.qza \
    --p-metric "observed_features" \
    --o-alpha-diversity obs-features-autofmt-contigs.qza
```
#### Linear Mixed Effects
```shell
qiime longitudinal linear-mixed-effects \
  --m-metadata-file sample-metadata.tsv obs-features-autofmt-contigs.qza \
  --p-state-column day-relative-to-fmt \
  --p-group-columns autoFmtGroup \
  --p-individual-id-column PatientID \
  --p-metric "observed_features" \
  --o-visualization lme-obs-features-treatmentVScontrol-contigs.qzv
```

### Beta Diversity
#### Jaccard Distance Matrix PCoA creation
```shell
qiime diversity beta \
  --i-table kraken2-autofmt-presence-absence-contigs.qza \
  --p-metric jaccard \
  --o-distance-matrix jaccard-autofmt-contigs.qza
```

```shell
qiime diversity pcoa \
  --i-distance-matrix jaccard-autofmt-contigs.qza \
  --o-pcoa jaccard-autofmt-pcoa-contigs.qza
```
#### Emperor Plot Creation
```shell
qiime emperor plot \
  --i-pcoa jaccard-autofmt-pcoa-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization jaccard-autofmt-emperor-contigs.qzv
```

```shell
qiime emperor plot \
  --i-pcoa jaccard-autofmt-pcoa-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-custom-axes week-relative-to-fmt \
  --o-visualization jaccard-autofmt-emperor-custom-contigs.qzv
```

### Taxa-bar Creation
```shell
qiime taxa barplot \
  --i-table kraken2-autofmt-presence-absence-contigs.qza \
  --i-taxonomy kraken2-taxonomy-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization taxa-bar-plot-autofmt-contigs.qzv
```
### Functional analysis
#### Obtaining Feature Table
```shell
wget -O eggnog-presence-absence-contigs.qza https://polybox.ethz.ch/index.php/s/YSXu2AaOFeusgRY/download
```
#### Filtering Feature Table
```shell
qiime feature-table filter-samples \
  --i-table eggnog-presence-absence-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table filtered-eggnog-presence-absence-contigs.qza
```
#### Jaccard Distance Matrix PCoA creation for gene diversity
```shell
qiime diversity beta \
  --i-table filtered-eggnog-presence-absence-contigs.qza \
  --p-metric jaccard \
  --o-distance-matrix jaccard-diamond-autofmt-contigs.qza 
```

```shell
qiime diversity pcoa \
  --i-distance-matrix jaccard-diamond-autofmt-contigs.qza  \
  --o-pcoa jaccard-diamond-autofmt-pcoa-contigs.qza
```
#### Emperor Plot Creation for gene diversity
```shell
qiime emperor plot \
  --i-pcoa  jaccard-diamond-autofmt-pcoa-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization jaccard-diamond-autofmt-pcoa-contigs.qzv
```

## MAG-based analysis
### BUSCO QC
```shell
wget -O busco-qc.qzv https://polybox.ethz.ch/index.php/s/fzAA003m6UVw5je/download
```
### Obtaining our Kraken2 reports
```shell
wget -O kraken2-reports-mags-derep.qza https://polybox.ethz.ch/index.php/s/n0L2vm16C1J6MHe/download
```

### Kraken2 annotation reports extraction
```shell
qiime tools extract  \
  --input-path  \
  --output-path  \
```


