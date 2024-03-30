# Metagenomics with QIIME 2 - Spring 2024 Leiden & Zurich Tutorial

```{warning}
Metagenomics analysis with QIIME 2 is in alpha release.
This means that results you generate should be considered preliminary, and NOT PUBLICATION QUALITY.
Additionally, interfaces are subject to change, and those changes may be backward incompatible (meaning that a command or file that works in one version of the QIIME 2 Shotgun Metagenomics distribution may not work in the next version of that distribution).
```

## Setup

Before we dive into the tutorial, let's talk about our workshop server directory structure.

### Directory structure
```shell
<your home directory>
└ workshop
   ├ reads
   ├ contigs
   └ sample-metadata.tsv
```
Let's verify that QIIME 2 is working by calling qiime.
```shell
qiime info
```
Before we start our analyses, we want to create sub-directories for each type of data we're examining to keep things organized. Within the `workshop` directory, we'll create the following sub-directories:
```
mkdir reads
```
```
mkdir contigs
```

````{toggle}
**Required databases**

In order to perform the taxonomic and functional annotation, we will need a couple of different reference databases. Below you will find instructions on how to download these databases using respective QIIME 2 actions.

1. Kraken 2/Bracken database

```shell
qiime moshpit build-kraken-db \
    --p-collection standard \
    --o-kraken2-database ./moshpit_tutorial/cache:kraken_standard \
    --o-bracken-database ./moshpit_tutorial/cache:bracken_standard \
    --verbose
```

2. EggNOG databases

```shell
qiime moshpit fetch-diamond-db \
    --o-diamond-db ./moshpit_tutorial/cache:eggnog_diamond_full \
    --verbose
```
```shell
qiime moshpit fetch-eggnog-db \
    --o-eggnog-db ./moshpit_tutorial/cache:eggnog_annot_full \
    --verbose
```

**Data retrieval from SRA**

The data we are using in this tutorial can be fetched from the SRA repository using the [q2-fondue](https://github.com/bokulich-lab/q2-fondue) plugin. We need to import the accession IDs into a QIIME 2 artifact:

```shell
qiime tools import \
  --type NCBIAccessionIDs \
  --input-path ids.tsv \
  --output-path ./moshpit_tutorial/ids.qza
```

We can then use the `get-sequences` action to download the data (please insert your e-mail address in the `--p-email` parameter):

```shell
qiime fondue get-sequences \
    --i-accession-ids ./moshpit_tutorial/ids.qza \
    --p-n-jobs 16 \
    --p-email you@tutorial.com \
    --o-single-reads ./moshpit_tutorial/cache:reads_single \
    --o-single-paired ./moshpit_tutorial/cache:reads_paired \
    --o-failed-runs ./moshpit_tutorial/cache:failed_runse \
    --verbose
```
This will download all the sequences into the QIIME 2 cache. It is a lot of data, so keep in mind that depending on your network speed, this might take a while.
````

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

````{note}
We will be using QIIME 2 View (view.qiime2.org) to examine our QIIME 2 visualizations. In order to do this, we first need to grab the URL for each visualization from the workshop server. For each visualization, you'll navigate to:
```shell
https://workshop-server.qiime2.org/[your-username]/
```
From here, you'll right click on the hyperlink associated with the visualization you'd like to view, copy the URL, navigate to QIIME 2 View and paste this URL under `a file from the web`. Note that you can also download any of these files directly from the server by left clicking on the hyperlink associated with the visualization you'd like to download.
````
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
    	--p-memory-mapping False ##set to False to shorten runtime
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
We are going to look at taxonomic annotations for our read-based analysis. In order to do that, we need to download our read table and taxonomy that we generated using Bracken.
```shell
wget -O ./reads/bracken-feature-table.qza https://polybox.ethz.ch/index.php/s/4Y1IGtZHTzo1KTi/download
```
```shell
wget -O ./reads/bracken-taxonomy.qza https://polybox.ethz.ch/index.php/s/haWDZzLcJsuiI9b/download
```
Before moving forward in our analysis, let's take a look at a visual summary of our feature table and re-examine our metadata summary. This will help to orient us with what our original data looks like, and put subsequent downstream steps in context.
```shell
qiime feature-table summarize \
  --i-table ./reads/bracken-feature-table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization ./reads/bracken-feature-table.qzv
```

### Feature Table Filtering
First, we need to remove samples that are not part of the autoFMT study from the feature table. We'll identify these samples using the metadata. Specifically, this step filters samples that do not contain a value in the autoFmtGroup column in the metadata.
```shell
qiime feature-table filter-samples \
  --i-table ./reads/bracken-feature-table.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table ./reads/bracken-autofmt-feature-table.qza
```

### Taxa Barplot Creation
Now we will use our filtered table and taxonomy to create a taxonomic barplot.
```shell
qiime taxa barplot \
  --i-table ./reads/bracken-autofmt-feature-table.qza \
  --i-taxonomy ./reads/bracken-taxonomy.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization ./reads/taxa-bar-plot-autofmt-reads.qzv
```
### Differential Abundance
#### Feature Table Preparation
ANCOM-BC cannot be run on repeated measures, so we need to filter to one timepoint. Here we will filter to the "peri" timepoint.
```shell
qiime feature-table filter-samples \
  --i-table ./reads/bracken-autofmt-feature-table.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where "[categorical-time-relative-to-fmt]='peri'" \
  --o-filtered-table ./reads/peri-fmt-table.qza
```
Now let's visualize our filtered table!
```shell
qiime feature-table summarize \
  --i-table ./reads/peri-fmt-table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization ./reads/peri-fmt-table.qzv
```
Upon examination of our filtered table, it looks like there is still a subject that has two samples at our peri timepoint. Let's filter that out!
```shell

echo SampleID > ./reads/samples-to-remove.tsv
echo SRR14092317 >> ./reads/samples-to-remove.tsv

```
```shell
qiime feature-table filter-samples \
  --i-table ./reads/peri-fmt-table.qza \
  --m-metadata-file ./reads/samples-to-remove.tsv \
  --p-exclude-ids \
  --o-filtered-table ./reads/id-filtered-peri-fmt-table.qza
```

```shell
qiime feature-table summarize \
  --i-table ./reads/id-filtered-peri-fmt-table.qza \
  --m-sample-metadata-file sample-metadata.tsv \
  --o-visualization ./reads/id-filtered-peri-fmt-table.qzv
```
Now we should collapse our table so our features are grouped at the species level.
```shell
qiime taxa collapse \
--i-table ./reads/id-filtered-peri-fmt-table.qza \
--i-taxonomy ./reads/bracken-taxonomy.qza \
--p-level 8 \
--o-collapsed-table ./reads/collapsed-8-id-filtered-peri-fmt-table.qza
```
#### Differential Abundance with ANCOM-BC
Now let's run ANCOM-BC! This is a great way to take a look at what features are either enriched or depleted, relative to a reference group of our choosing. Since we'd like to look at the differential abundance of the `autoFmtGroup`, we'll set our reference group to `control`.
```shell
 qiime composition ancombc \
  --i-table ./reads/collapsed-8-id-filtered-peri-fmt-table.qza  \
  --m-metadata-file sample-metadata.tsv \
  --p-formula autoFmtGroup \
  --p-reference-levels autoFmtGroup::control \
  --o-differentials ./reads/differentials-peri-autofmt.qza
```

```shell
qiime composition da-barplot \
  --i-data ./reads/differentials-peri-autofmt.qza \
  --p-significance-threshold 0.05 \
  --p-level-delimiter ";" \
  --o-visualization ./reads/differentials-peri-autofmt.qzv
```
## Contig-based analysis
````{toggle}
**Assemble Reads into Contigs with MEGAHIT**

The first step in recovering metagenome-assembled genomes (MAGs) is genome assembly itself. There are many genome assemblers available, two of which you can use through our QIIME 2 plugin - here, we will use MEGAHIT. MEGAHIT takes short DNA sequencing reads, constructs a simplified De Bruijn graph, and generates longer contiguous sequences called contigs, providing valuable genetic information for the next steps of our analysis.
```shell
qiime assembly assemble-megahit \
    --i-seqs "./moshpit_tutorial/cache:reads_no_host" \
    --p-presets "meta-sensitive" \
    --p-num-cpu-threads 64 \
    --p-num-partitions 4 \
    --o-contigs "./moshpit_tutorial/cache:contigs" \
    --verbose
```
**Contig QC with QUAST**

Once the reads are assembled into contigs, we can use QUAST to evaluate the quality of our assembly.
```shell
qiime assembly evaluate-contigs \
    --i-contigs "./moshpit_tutorial/cache:contigs" \
    --p-threads 128 \
    --p-memory-efficient \
    --o-visualization "./moshpit_tutorial/results/contigs.qzv" \
    --verbose
```
````
### QUAST QC
After assembling our reads into contigs, let's have a look at our contig quality metrics.
```shell
wget -O ./contigs/quast-qc.qzv https://polybox.ethz.ch/index.php/s/XyZfYkDEHh1nHZq/download
```

### Obtaining your Feature Table and Taxonomy Table

````{toggle}
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

Similar to our read-based analysis, we are now going to look at taxonomic annotations for our contig-based analysis. In order to do that, we need to download our contig feature table and taxonomy that we generated using Kraken2.

```shell
wget -O ./contigs/kraken2-presence-absence-contigs.qza https://polybox.ethz.ch/index.php/s/OYL590hv7eZJPWS/download
```
```shell
wget -O ./contigs/kraken2-taxonomy-contigs.qza https://polybox.ethz.ch/index.php/s/Wk0nsgQfEjgdabc/download
```
### Feature Table Filtering
Now that we have acquired our contig feature table, we will also remove samples that are not part of the autoFMT study (following the same approach we used in our read-based analysis).
```shell
qiime feature-table filter-samples \
  --i-table ./contigs/kraken2-presence-absence-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table ./contigs/kraken2-autofmt-presence-absence-contigs.qza
```
### Alpha Diversity
Here we'll examine and compare the community richness between our control and treatment groups.

#### Observed Features
To start with, we'll generate an `observed features` vector from our presence/absence feature table:
```shell
qiime diversity alpha \
    --i-table ./contigs/kraken2-autofmt-presence-absence-contigs.qza \
    --p-metric "observed_features" \
    --o-alpha-diversity ./contigs/obs-features-autofmt-contigs.qza
```
#### Linear Mixed Effects
We will use a linear mixed-effects model in order to manage the repeated measures in our dataset.
```shell
qiime longitudinal linear-mixed-effects \
  --m-metadata-file sample-metadata.tsv ./contigs/obs-features-autofmt-contigs.qza \
  --p-state-column day-relative-to-fmt \
  --p-group-columns autoFmtGroup \
  --p-individual-id-column PatientID \
  --p-metric "observed_features" \
  --o-visualization ./contigs/lme-obs-features-treatmentVScontrol-contigs.qzv
```

### Beta Diversity
Now that we better understand community richness trends, let's look at differences in microbial composition.

#### Jaccard Distance Matrix PCoA creation
Let's first create the Jaccard distance matrix!
```shell
qiime diversity beta \
  --i-table ./contigs/kraken2-autofmt-presence-absence-contigs.qza \
  --p-metric jaccard \
  --o-distance-matrix ./contigs/jaccard-autofmt-contigs.qza
```
Now, let's generate a PCoA from our Jaccard matrix.
```shell
qiime diversity pcoa \
  --i-distance-matrix ./contigs/jaccard-autofmt-contigs.qza \
  --o-pcoa ./contigs/jaccard-autofmt-pcoa-contigs.qza
```
#### Emperor Plot Creation
Now that we have our Jaccard diversity PCoA, let's visualize it!
```shell
qiime emperor plot \
  --i-pcoa ./contigs/jaccard-autofmt-pcoa-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization ./contigs/jaccard-autofmt-emperor-contigs.qzv
```
We can make `week-relative-to-fmt` a custom axis in our PCoA. This allows us to look at changes in microbial composition over the course of the study.
```shell
qiime emperor plot \
  --i-pcoa ./contigs/jaccard-autofmt-pcoa-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-custom-axes week-relative-to-fmt \
  --o-visualization ./contigs/jaccard-autofmt-emperor-custom-contigs.qzv
```

### Taxa Barplot Creation
We will now explore the microbial composition in our contig-based analysis by visualizing a taxonomic bar plot. Note that we are using a FeatureTable[PresenceAbsence], and thus are not talking about relative abundance.
```shell
qiime taxa barplot \
  --i-table ./contigs/kraken2-autofmt-presence-absence-contigs.qza \
  --i-taxonomy ./contigs/kraken2-taxonomy-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization ./contigs/taxa-bar-plot-autofmt-contigs.qzv
```
### Functional Analysis

````{toggle}
```shell
qiime moshpit eggnog-diamond-search \
  --i-sequences "./moshpit_tutorial/cache:contigs" \
  --i-diamond-db "./moshpit_tutorial/cache:eggnog_diamond_full"\
  --p-num-cpus 14 \
  --p-db-in-memory \
  --o-eggnog-hits "./moshpit_tutorial/cache:diamond_hits_contigs" \
  --o-table "./moshpit_tutorial/cache:diamond_feature_table_contigs" \
  --verbose
```
````

Here we will perform functional annotation of contigs to capture gene diversity!

#### Obtaining your Feature Table
You know the drill by now ;)
```shell
wget -O ./contigs/eggnog-presence-absence-contigs.qza https://polybox.ethz.ch/index.php/s/YSXu2AaOFeusgRY/download
```
#### Feature Table Filtering
Same here ;)
```shell
qiime feature-table filter-samples \
  --i-table ./contigs/eggnog-presence-absence-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table ./contigs/filtered-eggnog-presence-absence-contigs.qza
```
#### Jaccard Distance Matrix PCoA creation for gene diversity
We will again start by calculating our Jaccard beta diversity matrix.
```shell
qiime diversity beta \
  --i-table ./contigs/filtered-eggnog-presence-absence-contigs.qza \
  --p-metric jaccard \
  --o-distance-matrix ./contigs/jaccard-diamond-autofmt-contigs.qza
```
Then, we will generate our PCoA from Jaccard matrix.
```shell
qiime diversity pcoa \
  --i-distance-matrix ./contigs/jaccard-diamond-autofmt-contigs.qza  \
  --o-pcoa ./contigs/jaccard-diamond-autofmt-pcoa-contigs.qza
```
#### Emperor Plot Creation for gene diversity
Visualization time!
```shell
qiime emperor plot \
  --i-pcoa ./contigs/jaccard-diamond-autofmt-pcoa-contigs.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization ./contigs/jaccard-diamond-autofmt-pcoa-contigs.qzv
```

## Breakout Session
Before we begin the final segment of our analysis in this workshop, we're going to break off into two groups for a small interactive exercise! We will assign you to either group A or B, and will go through your respective exercises below.

### Group A
In this group exercise we will be examining the correlation between our Jaccard distance matrices for reads and contigs. We will not be providing commands for this, so do some poking around in the help text for commands within the diversity plugin to see what might make sense for comparing these two distance matrices. :)

````{dropdown} Hint!
Try running `qiime diversity --help` to view all of the available methods within this plugin. From here, you can run the `--help` command on any of the methods to learn more about their inputs and functionality (i.e. `qiime diversity beta --help`).
````

### Group B
In this group exercise we will be examining the correlation between our read-based Jaccard distance matrix and a read-based Bray-Curtis distance matrix. Note that you'll need to first generate the Bray-Curtis distance matrix before you can move forward with this comparison. We will not be providing commands for this, so do some poking around in the help text for commands within the diversity plugin to see what might make sense for comparing these two distance matrices. :)

````{dropdown} Hint!
Try running `qiime diversity --help` to view all of the available methods within this plugin. From here, you can run the `--help` command on any of the methods to learn more about their inputs and functionality (i.e. `qiime diversity beta --help`).
````

## MAG-based analysis
````{toggle}
Let's start binning our contigs into MAGs using various tools and methodologies!

**Read mapping**

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
**Binning**

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
This step generated a couple artifacts:

- `mags.qza`: these are our actual MAGS, per sample.
- `contig-map.qza`: this is a mapping between MAG IDs and IDs of contigs which belong to a given MAG.
- `unbinned-contigs.qza`: these are all the contigs that could not be assigned to any particular MAG.
From here, we will focus on the mags.qza artifact.

**MAGs QC with BUSCO**

BUSCO is used here to assess the completeness and quality of MAGs by searching for single-copy orthologous genes within the genomes.
```shell
qiime moshpit evaluate-busco \
    --i-bins "./moshpit_tutorial/cache:mags" \
    --p-lineage-dataset bacteria_odb10 \
    --p-cpu 196 \
    --o-visualization "./moshpit_tutorial/results/mags.qzv" \
    --verbose
```
````

### BUSCO QC
Here we will have a look at our BUSCO results to assess the completeness and quality of MAGs!
```shell
wget -O busco-qc.qzv https://polybox.ethz.ch/index.php/s/fzAA003m6UVw5je/download
```

````{toggle}
**MAGs dereplication**

Dereplication involves removing duplicate or nearly identical MAGs to reduce redundancy and improve downstream analyses. To dereplicate our MAGs, we will:

1. Compute hash sketches of every genome using [sourmash](https://sourmash.readthedocs.io/en/latest/) - you can think of these sketches as tiny representations of our genomes (sourmash compresses a lot of information into much smaller space).
2. Compare all of those sketches (genomes) to one another to generate a matrix of pairwise distances between our MAGs.
3. Dereplicate the genomes using the distance matrix and a fixed similarity threshold. The last action will simply choose the longest genome from all of the genomes belonging to the same cluster, given a similarity threshold.

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

**MAGs taxonomic annotation workflow**

This workflow focuses on annotating MAGs with taxonomic information using Kraken2, a tool for taxonomic classification. In this tutorial we will perfom taxonomic and functional annotation on dereplicated MAGs.

**MAGs classify with Kraken2**

MAGs are classified taxonomically using Kraken2, with parameters set for a confidence threshold and minimum base quality.

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
````

### Obtaining our Kraken2 reports

QIIME 2 does not stop you from using your favorite tools with its output! First, let's obtain an artifact containing Kraken 2 annotated MAGs from this dataset. We will visualize some of them with [pavian](https://fbreitwieser.shinyapps.io/pavian/).

```shell
wget -O kraken2-reports-mags-derep.qza https://polybox.ethz.ch/index.php/s/n0L2vm16C1J6MHe/download
```

### Kraken2 annotation reports export
Now, let's export this QIIME artifact and explore!

```shell
qiime tools export \
  --input-path kraken2-reports-mags-derep.qza \
  --output-path kraken2-reports-mags-derep
```

## Provenance Replay
Without looking back through all of the commands we ran in this tutorial, how many of you feel confident that you could re-run our analyses from memory? If you don't feel confident in that, you're not alone! It is very common to have difficulty remembering the exact commands you ran for a past analysis (or trying to figure out the commands that someone else ran from an external analysis). Even if you write down all of the steps you've taken, humans are fallible and our memories aren't perfect.

Each QIIME 2 Result (i.e. Artifact or Visualization) contains provenance that can be used as a reference to reconstruct the commands that were used to generate said result. Let's take a look at the provenance associated with one of the visualizations from our read-based analysis as an example!

While using provenance to manually reconstruct the commands used to generate a result is a reasonable workflow for one or two results, we need a more automated solution to reconstruct the commands for a larger analysis - such as the analyses we ran in this workshop. Luckily, provenance replay can handle this for us!

We'll start off by running provenance replay for all of the read-based results we've generated during this tutorial. We can run provenance replay on the entire `reads` directory. This will provide us with a replay supplement that contains all of the upstream commands used to generate each result, any relevant citations associated with each of the commands used (in BibTex format), and the recorded metadata used in each command.

```shell
qiime tools replay-supplement \
--in-fp ./reads \
--out-fp reads-replay-output
```

On your own, try generating the replay supplement for all of the contig-based results, and reconstruct a few of the commands used in that analysis!

**Hint**: use the `contigs` directory for your input filepath. :)
