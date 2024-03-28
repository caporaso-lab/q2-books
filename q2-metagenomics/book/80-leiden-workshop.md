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
  --o-filtered-table id-filtered-peri-fmt-table.qza
```

```shell
qiime feature-table summarize \
  --i-table id-filtered-peri-fmt-table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization id-filtered-peri-fmt-table.qzv
```

```shell
 qiime composition ancombc \
  --i-table id-filtered-peri-fmt-table.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-formula autoFmtGroup \
  --o-differentials differentials-peri-autofmt.qza
```
```shell
qiime composition da-barplot \
  --p-significance-threshold 0.05 \
  --o-visualization differentials-peri-autofmt.qzv
```
## Contig-based analysis
### QUAST QC
```shell
wget -O quast-qc.qzv https://polybox.ethz.ch/index.php/s/XyZfYkDEHh1nHZq/download
```
### Filtering Feature Table
```shell
qiime feature-table filter-samples \
  --i-table "./moshpit_tutorial/kraken_feature_table_contigs.qza" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table "./moshpit_tutorial/kraken_autofmt_feature_table_contigs.qza"
```
### Alpha Diversity
#### Observed Features 
```shell
qiime diversity alpha \
    --i-table "./moshpit_tutorial/kraken_autofmt_feature_table_contigs.qza" \
    --p-metric "observed_features" \
    --o-alpha-diversity "./moshpit_tutorial/obs_features_autofmt_contigs.qza"
```
#### Linear Mixed Effects
```shell
 qiime longitudinal linear-mixed-effects \
   --m-metadata-file "./moshpit_tutorial/metadata.tsv" "./moshpit_tutorial/obs_features_autofmt_contigs.qza" \
   --p-state-column DayRelativeToNearestHCT \
   --p-individual-id-column PatientID \
   --p-metric observed_features \
   --o-visualization "./moshpit_tutorial/results/lme_obs_features_HCT_contigs.qzv"
```

```shell
 qiime longitudinal linear-mixed-effects \
   --m-metadata-file "./moshpit_tutorial/metadata.tsv" "./moshpit_tutorial/obs_features_autofmt_contigs.qza" \
   --p-state-column day-relative-to-fmt \
   --p-individual-id-column PatientID \
   --p-metric observed_features \
   --o-visualization "./moshpit_tutorial/results/lme_obs_features_FMT_contigs.qzv"
```

```shell
qiime longitudinal linear-mixed-effects \
 --m-metadata-file "./moshpit_tutorial/metadata.tsv" "./moshpit_tutorial/obs_features_autofmt_contigs.qza" \
  --p-state-column day-relative-to-fmt \
  --p-group-columns autoFmtGroup \
  --p-individual-id-column PatientID \
  --p-metric observed_features \
  --o-visualization "./moshpit_tutorial/results/lme_obs_features_treatmentVScontrol_contigs.qzv"
```

### Beta Diversity
#### Jaccard Distance Matrix PCoA creation
```shell
qiime diversity beta \
  --i-table "./moshpit_tutorial/kraken_autofmt_feature_table_contigs.qza" \
  --p-metric jaccard \
  --o-distance-matrix "./moshpit_tutorial/jaccard_autofmt_contigs.qza"
```

```shell
qiime diversity pcoa \
  --i-distance-matrix "./moshpit_tutorial/jaccard_autofmt_contigs.qza" \
  --o-pcoa "./moshpit_tutorial/jaccard_autofmt_pcoa_contigs.qza"
```
#### Emperor Plot Creation
```shell
qiime emperor plot \
  --i-pcoa "./moshpit_tutorial/cache:jaccard_autofmt_pcoa_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --o-visualization "./moshpit_tutorial/results/jaccard_autofmt_emperor.qzv
```

```shell
qiime emperor plot \
  --i-pcoa "./moshpit_tutorial/cache:jaccard_autofmt_pcoa_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --p-custom-axes week-relative-to-fmt \
  --o-visualization "./moshpit_tutorial/results/jaccard_autofmt_emperor_custom.qzv"
```

### Taxa-bar Creation
```shell
qiime taxa barplot \
  --i-table "./moshpit_tutorial/cache:kraken_autofmt_feature_table_contigs" \
  --i-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --o-visualization "./moshpit_tutorial/results/taxa_bar_plot_autofmt_contigs.qzv"
```
### Functional analysis
### Filtering Feature Table
```shell
qiime feature-table filter-samples \
  --i-table "./moshpit_tutorial/cache:diamond_feature_table_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table "./moshpit_tutorial/cache:diamond_autofmt_feature_table_contigs"
```
#### Jaccard Distance Matrix PCoA creation for gene diversity
```shell
qiime diversity beta \
  --i-table "./moshpit_tutorial/cache:diamond_autofmt_feature_table_contigs" \
  --p-metric jaccard \
  --o-distance-matrix "./moshpit_tutorial/cache:jaccard_diamond_autofmt_contigs"
```

```shell
qiime diversity pcoa \
  --i-distance-matrix "./moshpit_tutorial/cache:jaccard_diamond_autofmt_contigs" \
  --o-pcoa "./moshpit_tutorial/cache:jaccard_diamond_autofmt_pcoa_contigs"
```
#### Emperor Plot Creation for gene diversity
```shell
qiime emperor plot \
  --i-pcoa  "./moshpit_tutorial/cache:jaccard_distance_matrix_pcoa_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --o-visualization "./moshpit_tutorial/results/jaccard_distance_matrix_pcoa_contigs.qzv"
```

## MAG-based analysis
### BUSCO QC
wget....

### Taxonomy table creation
```shell
qiime moshpit kraken2-to-mag-features \
 --i-reports "./moshpit_tutorial/cache:kraken_reports_derep_mags" \
 --i-hits "./moshpit_tutorial/cache:kraken_hits_derep_mags" \
 --o-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_derep_mags" \
 --verbose
```

#### Taxonomy table visualization
```shell
qiime metadata tabulate \
 --m-input-file "./moshpit_tutorial/cache:kraken_taxonomy_derep_mags" \
 --p-page-size "./moshpit_tutorial/cache:kraken_hits_derep_mags" \
 --verbose
```



