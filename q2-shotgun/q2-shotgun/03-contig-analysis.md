# Contig Analysis (Paula)
This section of the tutorial focuses on obtaining and analyzing contigs, which are contiguous sequences of DNA assembled from short reads obtained through sequencing techniques. Contigs are crucial in genome assembly and analysis.

## Assemble Reads into Contigs with MEGAHIT
The first step in recovering metagenome-assembled genomes (MAGs) is genome assembly itself. There are many genome assemblers available, two of which you can use through our QIIME 2 plugin - here, we will use MEGAHIT. MEGAHIT takes short DNA sequencing reads, constructs a simplified De Bruijn graph, and generates longer contiguous sequences called contigs, providing valuable genetic information for the next steps of our analysis.

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
Once the reads are assembled into contigs, we can use QUAST to evaluate the quality of our assembly. There are many metrics which can be used for that purpose but here we will focus on the two most popular metrics:

- **N50**: represents the contiguity of a genome assembly. It's defined as the length of the contig (or scaffold) at which 50% of the entire genome is covered by contigs of that length or longer - the higher this number, the better.
- **L50**: represents the number of contigs required to cover 50% of the genome's total length - the smaller this number, the better.
  
In addition to calculating generic statistics like N50 and L50, QUAST will try to identify potential genomes from which the analyzed contigs originated. Alternatively, we can provide it with a set of reference genomes we would like it to run the analysis against using `--i-references`.
```bash
qiime assembly evaluate-contigs \
    --i-contigs "./moshpit_tutorial/cache:contigs" \
    --p-threads 128 \
    --p-memory-efficient \
    --o-visualization "./moshpit_tutorial/results/contigs.qzv" \
    --verbose
```
## Contig Taxonomic Annotation Workflow
Now we are ready to perform taxonomic classification of our contigs. 
  
### Classify Contigs with Kraken2
Here, we are focusing on [Kraken 2](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-019-1891-0) - one of the most popular taxonomic classifiers for metagenomic data. 
Kraken 2 requires a pre-built database, which we have already built on the `01-sra-data-access` section of this tutorial, so that it can compare the analyzed genomes to a reference. In this example, we are using the [Standard database](https://benlangmead.github.io/aws-indexes/k2), which is a database built using all archaeal, bacterial, viral, plasmid and human sequences found in the NCBI's RefSeq database. Since Kraken 2 classification is based on comparing k-mer profiles, this database contains pre-calculated k-mer profiles for all the genomes listed earlier and stored in the so-called \"hash tables\" - data structres optimized for efficient data storage and retrieval. Alternatively, you can also use `qiime moshpit classify-kaiju` to classify your contigs with [Kaiju](https://github.com/bioinformatics-centre/kaiju).
- The `--p-confidence` and `--p-minimum-base-quality` are deviations from kraken's defaults.
- The database used here is the `Standard` database, defined [here](https://benlangmead.github.io/aws-indexes/k2).
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
With this previous action we got two new artifacts: FeatureData[Kraken2Report % Properties('contigs')] and FeatureData[Kraken2Output % Properties('contigs')]. The first one contains the Kraken 2 report: a tree-like representation of all the identified taxa. The second one is a list of all contigs with their corresponding identified taxa. 

### Presence/Absence Feature Table Creation
A natural next step would now be to estimate the relative frequencies of those taxa in our samples, however this is not yet possible to do on contigs with QIIME 2 (coming soon though!) Therefore, to convert those into a more QIIME-like taxonomy, run the following action:
```bash
qiime moshpit kraken2-to-features \
  --i-reports "./moshpit_tutorial/cache:kraken_reports_contigs" \
  --o-table "./moshpit_tutorial/cache:kraken_feature_table_contigs" \
  --o-taxonomy "./moshpit_tutorial/cache:kraken_taxonomy_contigs" \
  --verbose
```
## Filtering Feature Table and Normalization
Once we have feature table, this is becomes alot more similar to the amplicon workflow of QIIME 2. 

In this tutorial, we’re going to work specifically with samples that were included in the autoFMT randomized trial. Many of these subjects dropped out before randomization (placing the subject into FMT group or Control group) and therefore do not have a value in the `autoFmtGroup`. 

We need to filter our feature table to contain samples that were in the autoFMT study by filtering out any samples that are null in the metadata column autoFmtGroup. 

```bash
qiime feature-table filter-samples \
  --i-table "./moshpit_tutorial/cache:kraken_feature_table_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table "./moshpit_tutorial/cache:kraken_autofmt_feature_table_contigs"
```
## Alpha Diversity on Presence/Absence Feature Table
First we'll look for general patterns, by comparing different categorical groupings of samples to see if there is some relationship to richness.

To start with, we'll generate an 'observed features' vector from our presence/absence feature table:

```bash
qiime diversity alpha \
    --i-table "./moshpit_tutorial/cache:kraken_autofmt_feature_table_contigs" \
    --p-metric "observed_features" \
    --o-alpha-diversity "./moshpit_tutorial/cache:obs_features_autofmt_contigs"
```
### Linear Mixed Effects
In order to manage the repeated measures, we will use a linear
mixed-effects model. In a mixed-effects model, we combine fixed-effects (your typical linear regression coefficients) with random-effects. These random
effects are some (ostensibly random) per-group coefficient which minimizes the
error within that group. In our situation, we would want our random effect to
be the `PatientID` as we can see each subject has a different baseline for
richness (and we have multiple measures for each patient).
By making that a random effect, we can more accurately ascribe associations to
the fixed effects as we treat each sample as a "draw" from a per-group
distribution.

There are several ways to create a linear model with random effects, but we
will be using a *random-intercept*, which allows for the per-subject intercept
to take on a different average from the population intercept (modeling what we
saw in the group-significance plot above).

First let's evaluate the general trend of the Bone Marrow transplant.

```bash
 qiime longitudinal linear-mixed-effects \
   --m-metadata-file "./moshpit_tutorial/metadata.tsv" "./moshpit_tutorial/cache:obs_features_autofmt_contigs" \
   --p-state-column DayRelativeToNearestHCT \
   --p-individual-id-column PatientID \
   --p-metric observed_features \
   --o-visualization "./moshpit_tutorial/results/lme_obs_features_HCT_contigs.qzv"
```

We may also be interested in the effect of the auto fecal microbiota transplant. It should be known that these are generally correlated, so choosing one model over the other will require external knowledge.

```bash
 qiime longitudinal linear-mixed-effects \
   --m-metadata-file "./moshpit_tutorial/metadata.tsv" "./moshpit_tutorial/cache:obs_features_autofmt_contigs" \
   --p-state-column day-relative-to-fmt \
   --p-individual-id-column PatientID \
   --p-metric observed_features \
   --o-visualization "./moshpit_tutorial/results/lme_obs_features_FMT_contigs.qzv"
```

We also see a downward trend from the FMT. Since the goal of the FMT was to
ameliorate the impact of the bone marrow transplant protocol (which involves
an extreme course of antibiotics) on gut health, and the timing of the FMT is
related to the timing of the marrow transplant, we might deduce that the
negative coefficient is primarily related to the bone marrow transplant
procedure. (We can't prove this with statistics alone however, in this case,
we are using abductive reasoning).

Looking at the log-likelihood, we also note that the HCT result is slightly
better than the FMT in accounting for the loss of richness. But only slightly,
if we were to perform model testing it may not prove significant.

In any case, we can ask a more targeted question to identify if the FMT was
useful in recovering richness.

By adding the ``autoFmtGroup`` to our linear model, we can see if there
are different slopes for the two groups, based on an *interaction term*.

```bash
qiime longitudinal linear-mixed-effects \
 --m-metadata-file "./moshpit_tutorial/metadata.tsv" "./moshpit_tutorial/cache:obs_features_autofmt_contigs" \
  --p-state-column day-relative-to-fmt \
  --p-group-columns autoFmtGroup \
  --p-individual-id-column PatientID \
  --p-metric observed_features \
  --o-visualization "./moshpit_tutorial/results/lme_obs_features_treatmentVScontrol_contigs.qzv"
```

## Beta Diversity 
Now that we better understand community richness trends, lets look at differences in microbial composition.

### Jaccard Distance Matrix PCoA creation
Let's first create the Jaccard distance matrix. 
```bash
qiime diversity beta \
  --i-table "./moshpit_tutorial/cache:kraken_autofmt_feature_table_contigs" \
  --p-metric jaccard \
  --o-distance-matrix "./moshpit_tutorial/cache:jaccard_autofmt_contigs"
```
Now, let's generate a PCoA from Jaccard matrix.
```bash
qiime diversity pcoa \
  --i-distance-matrix "./moshpit_tutorial/cache:jaccard_autofmt_contigs" \
  --o-pcoa "./moshpit_tutorial/cache:jaccard_autofmt_pcoa_contigs"
```
### Emperor Plot Creation
Now that we have our Jaccard diversity PCoA, lets visualize it!
```bash
qiime emperor plot \
  --i-pcoa pcoa-braycurtis-auto-fmt.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --o-visualization braycurtis-auto-fmt-emperor.qzv 
```
We can make ` week-relative-to-fmt ` a custom axis in our PCOA. This allows us to look at changes in microbial composition over the couse of the study.  
```bash
qiime emperor plot \
  --i-pcoa pcoa-braycurtis-auto-fmt.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --p-custom-axes week-relative-to-fmt \
  --o-visualization braycurtis-auto-fmt-emperor-custom.qzv 
```

## Taxa-bar Creation
Another way we can look at microbial composition is to investigate the taxa barplot. One thing to Note, these tend to be even more chaotic then the Amplicon data. 
```bash
qiime taxa barplot \
  --i-table table-bracken.qza \
  --i-taxonomy taxonomy-bracken.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --o-visualization taxa-bar-plot.qzv
```

Lets highlight some features of metagenomic data that we wouldn't see in Amplicon data.

### Viral Taxa-bar Creation
We will take a peek at the viral community members.

First, we filter down to the features taxomically labeled "Virus" 

```bash
qiime taxa filter-table \
  --i-table autofmt-filt-table.qza \
  --i-taxonomy taxonomy-bracken.qza \
  --p-include Viruses \
  --o-filtered-table virus-autofmt-table
```

Then we will filtered out any samples that were below 1,000.  
```bash
qiime feature-table filter-samples \
  --i-table virus-autofmt-table.qza \
  --p-min-frequency 1240 \
  --o-filtered-table autofmt-virus-feature-contingency-filtered-table.qza
```
Now we can make our Viral Taxa Bar plot 
```bash
qiime taxa barplot \
  --i-table  autofmt-virus-feature-contingency-filtered-table.qza \
  --i-taxonomy taxonomy-bracken.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --o-visualization virus-taxa-bar-plot.qzv
```

## Contig functional annotation workflow
Here we will perform functional annotation of contigs to capture gene diversity.
  
### EggNOG search using diamond aligner
Searches for homologous sequences in the EggNOG database using the Diamond aligner for faster processing.
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
Calculates gene diversity metrics, specifically Jaccard distance, for contigs.
Calculate Jaccard beta-diversity matrix
```bash
qiime diversity beta \
  --i-table "./moshpit_tutorial/cache:diamond_feature_table_contigs" \
  --p-metric jaccard \
  --o-distance-matrix "./moshpit_tutorial/cache:jaccard_distance_matrix_contigs"
```
Generate a PCoA from Jaccard matrix
```bash
qiime diversity pcoa \
  --i-distance-matrix "./moshpit_tutorial/cache:jaccard_distance_matrix_contigs" \
  --o-pcoa "./moshpit_tutorial/cache:jaccard_distance_matrix_pcoa_contigs"
```
Visualize PCoA using Emperor
```bash
qiime emperor plot \
  --i-pcoa  "./moshpit_tutorial/cache:jaccard_distance_matrix_pcoa_contigs" \
  --m-metadata-file "./moshpit_tutorial/metadata.tsv" \
  --o-visualization "./moshpit_tutorial/results/jaccard_distance_matrix_pcoa_contigs.qzv"
```

### Annotate orthologs against eggNOG database
Annotates contigs with functional information from the EggNOG database.
```bash
qiime moshpit eggnog-annotate \
 --i-eggnog-hits "./moshpit_tutorial/cache:contigs" \
 --i-eggnog-db "./moshpit_tutorial/cache:eggnog_annot_full" \
 --o-ortholog-annotations "./moshpit_tutorial/cache:eggnog_annotated_contigs" \
 --verbose
```

