# Metagenomics with QIIME 2 - Leiden 2024

## Setup

Before we dive into the tutorial, let's set up the required directory structre and make sure we have all the required software installed.

### QIIME 2 metagenome distribution

You can install the latest distribution of the QIIME 2 metagenome distribution by following the instructions [here](https://docs.qiime2.org/2024.2/install/native/#qiime-2-metagenome-distribution). Once installed, you can activate the environment by running the following command:

```shell
conda activate qiime2-shotgun-2024.2
```

### Directory structure

Below you can see the directory structure that we will use throughout this tutorial:

```shell
<your working directory>
├── moshpit_tutorial
│   ├── cache
│   ├── results
```

Once you decided on the location of your working directory, let's create the `results` subdirectory by running the following command:

```shell
mkdir -p moshpit_tutorial/results
```

Next, we create the `cache` subdirectory (this is where majority of the data will be written to by QIIME 2) by running the following command:

```shell
qiime tools cache-create \
  --cache ./moshpit_tutorial/cache
```

We will be saving all the artifacts into that QIIME cache and all the final visualizations and tables into the `results` directory. If you want to read more about the QIIME cache, you can do so [here](https://dev.qiime2.org/latest/api-reference/cache/).

## Read-based analysis
### Filtering Feature Table
```shell
qiime feature-table filter-samples \
  --i-table "./moshpit_tutorial/cache:kraken_feature_table_reads" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table "./moshpit_tutorial/cache:kraken_autofmt_feature_table_reads"
```

### Taxa-bar Creation
```shell
qiime taxa barplot \
  --i-table "./moshpit_tutorial/cache:kraken_autofmt_feature_table_reads" \
  --i-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_reads" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv"v \
  --o-visualization "./moshpit_tutorial/results/taxa_bar_plot_autofmt_reads.qzv"
```

## Contig-based analysis
### Filtering Feature Table
```shell
qiime feature-table filter-samples \
  --i-table "./moshpit_tutorial/cache:kraken_feature_table_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table "./moshpit_tutorial/cache:kraken_autofmt_feature_table_contigs"
```
### Alpha Diversity
#### Observed Features 
```shell
qiime diversity alpha \
    --i-table "./moshpit_tutorial/cache:kraken_autofmt_feature_table_contigs" \
    --p-metric "observed_features" \
    --o-alpha-diversity "./moshpit_tutorial/cache:obs_features_autofmt_contigs"
```
#### Linear Mixed Effects
```shell
 qiime longitudinal linear-mixed-effects \
   --m-metadata-file "./moshpit_tutorial/metadata.tsv" "./moshpit_tutorial/cache:obs_features_autofmt_contigs" \
   --p-state-column DayRelativeToNearestHCT \
   --p-individual-id-column PatientID \
   --p-metric observed_features \
   --o-visualization "./moshpit_tutorial/results/lme_obs_features_HCT_contigs.qzv"
```

```shell
 qiime longitudinal linear-mixed-effects \
   --m-metadata-file "./moshpit_tutorial/metadata.tsv" "./moshpit_tutorial/cache:obs_features_autofmt_contigs" \
   --p-state-column day-relative-to-fmt \
   --p-individual-id-column PatientID \
   --p-metric observed_features \
   --o-visualization "./moshpit_tutorial/results/lme_obs_features_FMT_contigs.qzv"
```

```shell
qiime longitudinal linear-mixed-effects \
 --m-metadata-file "./moshpit_tutorial/metadata.tsv" "./moshpit_tutorial/cache:obs_features_autofmt_contigs" \
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
  --i-table "./moshpit_tutorial/cache:kraken_autofmt_feature_table_contigs" \
  --p-metric jaccard \
  --o-distance-matrix "./moshpit_tutorial/cache:jaccard_autofmt_contigs"
```

```shell
qiime diversity pcoa \
  --i-distance-matrix "./moshpit_tutorial/cache:jaccard_autofmt_contigs" \
  --o-pcoa "./moshpit_tutorial/cache:jaccard_autofmt_pcoa_contigs"
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
#### Jaccard Distance Matrix PCoA creation for gene diversity
```shell
qiime diversity beta \
  --i-table "./moshpit_tutorial/cache:diamond_feature_table_contigs" \
  --p-metric jaccard \
  --o-distance-matrix "./moshpit_tutorial/cache:jaccard_distance_matrix_contigs"
```

```shell
qiime diversity pcoa \
  --i-distance-matrix "./moshpit_tutorial/cache:jaccard_distance_matrix_contigs" \
  --o-pcoa "./moshpit_tutorial/cache:jaccard_distance_matrix_pcoa_contigs"
```
#### Emperor Plot Creation for gene diversity
```shell
qiime emperor plot \
  --i-pcoa  "./moshpit_tutorial/cache:jaccard_distance_matrix_pcoa_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --o-visualization "./moshpit_tutorial/results/jaccard_distance_matrix_pcoa_contigs.qzv"
```

## MAG-based analysis
#### Taxonomy table creation
```shell
qiime moshpit kraken2-to-mag-features \
 --i-reports "./moshpit_tutorial/cache:kraken_reports_derep_mags" \
 --i-hits "./moshpit_tutorial/cache:kraken_hits_derep_mags" \
 --o-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_derep_mags" \
 --verbose
```

### Taxonomy table visualization
```shell
qiime metadata tabulate \
 --m-input-file "./moshpit_tutorial/cache:kraken_taxonomy_derep_mags" \
 --p-page-size "./moshpit_tutorial/cache:kraken_hits_derep_mags" \
 --verbose
```



