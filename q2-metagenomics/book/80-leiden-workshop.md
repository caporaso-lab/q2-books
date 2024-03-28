# Metagenomics with QIIME 2 - Leiden 2024

## Setup

Before we dive into the tutorial, let's set up the required directory structure and make sure we have all the required software installed.

### Directory structure
```shell
<your home directory>
└ workshop
```

## Metadata
First, let's grab our sample metadata! 
```shell
wget -O sample-metadata.tsv https://polybox.ethz.ch/index.php/s/79s2cQry8Ll0FGq/download
```
Next, let's take a look at the metadata by visualizing it in QIIME 2.
```shell
qiime metadata tabulate \
  --m-input-file sample-metadata.tsv \
  --o-visualization sample-metadata.qzv
```

```{note}
We will be using QIIME 2 View (view.qiime2.org) to examine our QIIME 2 visualizations. In order to do this, we first need to download each visualization from the workshop server. For each visualization, you'll navigate to:
```
https://workshop-server.qiime2.org/<your-user-name>/
```
From here, you'll click on the hyperlink associated with the visualization you'd like to download.
```
## Read-based analysis
````{toggle}
```shell
qiime moshpit classify-kraken2 \
	--i-seqs ./moshpit_tutorial/cache:workshop-reads \
	--i-kraken2-db ./moshpit_tutorial/cache:kracken_standard \
	--p-threads 40 \
	--p-confidence 0.6 \
	--p-minimum-base-quality 20 \
	--o-hits ./moshpit_tutorial/cache:workshop_kraken_db_hits \
	--o-reports ./moshpit_tutorial/cache:workshop_kraken_db_reports \
	--p-report-minimizer-data \
	--use-cache ./moshpit_tutorial/cache \
	--parallel-config slurm_config.toml \
    	--verbose \
    	--p-memory-mapping False ##this was taking too much time for me
```

```shell
qiime moshpit estimate-bracken \
    --i-bracken-db ./moshpit_tutorial/cache:bracken_standard \
    --p-read-len 100 \
    --i-kraken-reports ./moshpit_tutorial/cache:workshop_kraken_db_reports \
    --o-reports ./moshpit_tutorial/kraken-outputs/bracken-reports.qza \
    --o-taxonomy ./moshpit_tutorial/kraken-outputs/taxonomy-bracken.qza \
    --o-table ~./moshpit_tutorial/kraken-outputs/table-bracken.qza
```

````
### Obtaining your Feature Table and Taxonomy Table
We are going to look at taxonomic annotations for our read based analysis. In order to do that, we need to download our read table and taxonomy that we generated using Bracken. 
```shell
wget -O bracken-feature-table.qza https://polybox.ethz.ch/index.php/s/4Y1IGtZHTzo1KTi/download
```
```shell
wget -O bracken-taxonomy.qza https://polybox.ethz.ch/index.php/s/haWDZzLcJsuiI9b/download
```
### Filtering Feature Table
First, we’ll remove samples that are not part of the autoFMT study from the feature table. We identify these samples using the metadata. Specifically, this step filters samples that do not contain a value in the autoFmtGroup column in the metadata.
```shell
qiime feature-table filter-samples \
  --i-table bracken-feature-table.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table bracken-autofmt-feature-table.qza
```

### Taxa Barplot Creation
Now we will use our table and taxonomy to create a taxa barplot.
```shell 
qiime taxa barplot \
  --i-table bracken-autofmt-feature-table.qza \
  --i-taxonomy bracken-taxonomy.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization taxa-bar-plot-autofmt-reads.qzv
```
### Differential abundance
#### Feature Table preparation
ANCOM-BC can not be run on repeated measures. So we will need to filter to one timepoint. Here we filter to the "peri" timepoint. 
```shell
qiime feature-table filter-samples \
  --i-table bracken-autofmt-feature-table.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where "[categorical-time-relative-to-fmt]='peri'" \
  --o-filtered-table peri-fmt-table.qza
```
Let's visualize our filtered table! 
```shell
qiime feature-table summarize \
  --i-table peri-fmt-table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization peri-fmt-table.qzv
```
Looks like there is still a subject that has two samples at our peri timepoint. Let's filter that out! 
```shell

echo SampleID > samples-to-remove.tsv
echo SRR14092317 >> samples-to-remove.tsv

```
```shell
qiime feature-table filter-samples \
  --i-table peri-fmt-table.qza \
  --m-metadata-file samples-to-remove.tsv \
  --p-exclude-ids \
  --o-filtered-table id-filtered-peri-fmt-table.qza
```

```shell
qiime feature-table summarize \
  --i-table id-filtered-peri-fmt-table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization id-filtered-peri-fmt-table.qzv
```
Now, we should collapse our table so our features are grouped at the species level. 
```shell
qiime taxa collapse \
--i-table id-filtered-peri-fmt-table.qza \
--i-taxonomy bracken-taxonomy.qza \
--p-level 8 \
--o-collapsed-table collapsed-8-id-filtered-peri-fmt-table.qza 
```
#### ANCOM-BC
Now, we run ANCOM-BC!
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
````{toggle}
```shell
qiime assembly assemble-megahit \
    --i-seqs "./moshpit_tutorial/cache:reads_no_host" \
    --p-presets "meta-sensitive" \
    --p-num-cpu-threads 64 \
    --p-num-partitions 4 \
    --o-contigs "./moshpit_tutorial/cache:contigs" \
    --verbose
```

```shell
qiime assembly evaluate-contigs \
    --i-contigs "./moshpit_tutorial/cache:contigs" \
    --p-threads 128 \
    --p-memory-efficient \
    --o-visualization "./moshpit_tutorial/results/contigs.qzv" \
    --verbose
```

```shell
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

```shell
qiime moshpit kraken2-to-features \
  --i-reports "./moshpit_tutorial/cache:kraken_reports_contigs" \
  --o-table "./moshpit_tutorial/cache:kraken_feature_table_contigs" \
  --o-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_contigs" \
  --verbose
```

````


### QUAST QC
After assembling our reads into contigs, let's have a look at our contig quality metrics.
```shell
wget -O quast-qc.qzv https://polybox.ethz.ch/index.php/s/XyZfYkDEHh1nHZq/download
```
### Obtaining your Feature Table and Taxonomy Table
Similarly as we did for reads, we are now going to look at taxonomic annotations for our contigs based analysis. In order to do that, we need to download our contig feature table and taxonomy that we generated using Kraken2. 

```shell
wget -O kraken2-presence-absence-contigs.qza https://polybox.ethz.ch/index.php/s/OYL590hv7eZJPWS/download
```
```shell
wget -O kraken2-taxonomy-contigs.qza https://polybox.ethz.ch/index.php/s/Wk0nsgQfEjgdabc/download
```
### Filtering Feature Table
Now that we have acquaired our contig feature table, we will also remove samples that are not part of the autoFMT study following the exact same approach as we did for the reads. 
```shell
qiime feature-table filter-samples \
  --i-table kraken2-presence-absence-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table kraken2-autofmt-presence-absence-contigs.qza
```
### Alpha Diversity
Here we'll look and compare community richness between our control and treatment group.

#### Observed Features 
To start with, we'll generate an 'observed features' vector from our presence/absence feature table:
```shell
qiime diversity alpha \
    --i-table kraken2-autofmt-presence-absence-contigs.qza \
    --p-metric "observed_features" \
    --o-alpha-diversity obs-features-autofmt-contigs.qza
```
#### Linear Mixed Effects
We will use a linear mixed-effects model in order to manage the repeated measures in our dataset.
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
Now that we better understand community richness trends, let's look at differences in microbial composition.

#### Jaccard Distance Matrix PCoA creation
Let's first create the Jaccard distance matrix!
```shell
qiime diversity beta \
  --i-table kraken2-autofmt-presence-absence-contigs.qza \
  --p-metric jaccard \
  --o-distance-matrix jaccard-autofmt-contigs.qza
```
Now, let's generate a PCoA from Jaccard matrix.
```shell
qiime diversity pcoa \
  --i-distance-matrix jaccard-autofmt-contigs.qza \
  --o-pcoa jaccard-autofmt-pcoa-contigs.qza
```
#### Emperor Plot Creation
Now that we have our Jaccard diversity PCoA, let's visualize it!
```shell
qiime emperor plot \
  --i-pcoa jaccard-autofmt-pcoa-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization jaccard-autofmt-emperor-contigs.qzv
```
We can make week-relative-to-fmt a custom axis in our PCoA. This allows us to look at changes in microbial composition over the course of the study.
```shell
qiime emperor plot \
  --i-pcoa jaccard-autofmt-pcoa-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-custom-axes week-relative-to-fmt \
  --o-visualization jaccard-autofmt-emperor-custom-contigs.qzv
```

### Taxa-bar Creation
We will now explore our contig microbial composition by visualizing a taxa bar plot. Note that we are using a FeatureTable[PresenceAbsence], hence we are not talking about relative abundance in this case.
```shell
qiime taxa barplot \
  --i-table kraken2-autofmt-presence-absence-contigs.qza \
  --i-taxonomy kraken2-taxonomy-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization taxa-bar-plot-autofmt-contigs.qzv
```
### Functional analysis
Here we will perform functional annotation of contigs to capture gene diversity!
#### Obtaining Feature Table
You know the drill by now ;)
```shell
wget -O eggnog-presence-absence-contigs.qza https://polybox.ethz.ch/index.php/s/YSXu2AaOFeusgRY/download
```
#### Filtering Feature Table
Same here ;)
```shell
qiime feature-table filter-samples \
  --i-table eggnog-presence-absence-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table filtered-eggnog-presence-absence-contigs.qza
```
#### Jaccard Distance Matrix PCoA creation for gene diversity
We will again start by calculating our Jaccard beta-diversity matrix.
```shell
qiime diversity beta \
  --i-table filtered-eggnog-presence-absence-contigs.qza \
  --p-metric jaccard \
  --o-distance-matrix jaccard-diamond-autofmt-contigs.qza 
```
Then, we will generate our PCoA from Jaccard matrix.
```shell
qiime diversity pcoa \
  --i-distance-matrix jaccard-diamond-autofmt-contigs.qza  \
  --o-pcoa jaccard-diamond-autofmt-pcoa-contigs.qza
```
#### Emperor Plot Creation for gene diversity
Visulization time!
```shell
qiime emperor plot \
  --i-pcoa  jaccard-diamond-autofmt-pcoa-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization jaccard-diamond-autofmt-pcoa-contigs.qzv
```

## MAG-based analysis
### BUSCO QC
Here we will have a look at our BUSCO results to assess the completeness and quality of MAGs!
```shell
wget -O busco-qc.qzv https://polybox.ethz.ch/index.php/s/fzAA003m6UVw5je/download
```
### Obtaining our Kraken2 reports
QIIME 2 does not stop you from using other tools with its output! First, let's obtain an artifact containing Kraken 2 annotated MAGs from this dataset.
```shell
wget -O kraken2-reports-mags-derep.qza https://polybox.ethz.ch/index.php/s/n0L2vm16C1J6MHe/download
```

### Kraken2 annotation reports extraction
Now, let's unzip this QIIME artifact and explore!
```shell
qiime tools extract \
  --input-path kraken2-reports-mags-derep.qza \
  --output-path kraken2-reports-mags-derep.txt \
```


