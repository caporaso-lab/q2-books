# Taxonomic annotation of reads (Chloe)

```bash
qiime moshpit classify-kraken2 \
	--i-seqs /scratch/crh423/cache:workshop-reads \
	--i-kraken2-db /scratch/crh423/cache:workshop-kraken-db \
	--p-threads 40 \
	--p-confidence 0.6 \
	--p-minimum-base-quality 20 \
	--o-hits /scratch/crh423/cache:workshop_kraken_db_hits \
	--o-reports /scratch/crh423/cache:workshop_kraken_db_reports \
	--p-report-minimizer-data \
	--use-cache /scratch/crh423/cache \
	--parallel-config ~/shotgun-test/slurm_config.toml \
    --verbose \
    --p-memory-mapping False ##this was taking too much time for me 
```
```bash
qiime moshpit estimate-bracken \
    --i-bracken-db /projects/microbiome/biological-reference-data/2023.06.05-k2-plus-pf-bracken-db.qza \
    --p-read-len 100 \
    --i-kraken-reports /scratch/crh423/cache:workshop_kraken_db_reports \
    --o-reports ~/chloe-analysis/sra-shotgun-workshop/kraken-outputs/bracken-reports.qza \
    --o-taxonomy ~/chloe-analysis/sra-shotgun-workshop/kraken-outputs/taxonomy-bracken.qza \
    --o-table ~/chloe-analysis/sra-shotgun-workshop/kraken-outputs/table-bracken.qza
```

```bash
qiime feature-table filter-samples \
  --i-table table-bracken.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --p-where 'autoFmtGroup IS NOT NULL' \
  --o-filtered-table autofmt-table.qza
```

```bash
qiime feature-table relative-frequency \
    --i-table autofmt-table.qza \
	--o-relative-frequency-table autofmt-table-rf.qza \
```

# Alpha diversity 


First we'll look for general patterns, by comparing different categorical groupings of samples to see if there is some relationship to richness.

To start with, we'll gernate an 'observed features' vector from our relative frequency table:

```bash
qiime diversity alpha \
    --i-table autofmt-table-rf.qza \
    --p-metric "observed_features" \
    --o-alpha-diversity obs-autofmt-bracken-rf
```
The first thing to notice is the high variability in each individual's richness
(`PatientID`). The centers and spreads of the individual distributions are
likely to obscure other effects, so we will want to keep this in mind.
Additionally, we have repeated measures of each individual, so we are
violating independence assumptions when looking at other categories.
(Kruskal-Wallis is a non-parameteric test, but like most tests, still requires
samples to be independent.)

Keeping in mind that other categories are probably inconclusive,
we notice that there are (amusingly, and somewhat reassuringly) differences
in stool consistency (solid vs non-solid).

Because these data were derived from a study in which participants recieved
auto-fecal microbiota transplant, we may also be interested in whether there
was a difference in richness between the control group and the auto-FMT goup.

Looking at ``autoFmtGroup`` we see that there is no apparent difference, but
we also know that we are violating independence with our repeated measures, and
all patients recieved a bone-marrow transplant which may be a stronger effect.
(The goal of the auto-FMT was to mitigate the impact of the marrow transplant.)

We will use a more advanced statistical model to explore this question.
```bash
qiime diversity alpha-group-significance \
    --i-alpha-diversity obs-table-bracken-rf.qza \
    --m-metadata-file ../new-sample-metadata.tsv \
    --o-visualization obs-table-bracken-rf-group-sig.qzv
```

## Linear Mixed Effects

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
   --m-metadata-file ../new-sample-metadata.tsv obs-autofmt-bracken-rf.qza \
   --p-state-column DayRelativeToNearestHCT \
   --p-individual-id-column PatientID \
   --p-metric observed_features \
   --o-visualization lme-obs-features-HCT.qzv
```

Here we see a significant association between richness and the bone marrow
transplant.

We may also be interested in the effect of the auto fecal microbiota transplant. It should be known that these are generally correlated, so choosing one model over the other will require external knowledge.

```bash
qiime longitudinal linear-mixed-effects \
  --m-metadata-file ../new-sample-metadata.tsv obs-autofmt-bracken-rf.qza \
  --p-state-column day-relative-to-fmt \
  --p-individual-id-column PatientID \
  --p-metric observed_features \
  --o-visualization lme-obs-features-FMT.qzv
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
  --m-metadata-file ../new-sample-metadata.tsv obs-autofmt-bracken-rf.qza \
  --p-state-column day-relative-to-fmt \
  --p-group-columns autoFmtGroup \
  --p-individual-id-column PatientID \
  --p-metric observed_features \
  --o-visualization lme-obs-features-treatmentVScontrol.qzv
```
Here we see that the ``autoFmtGroup`` is not on its own a significant predictor
of richness, but and no significance in it's interaction term with ``Q('day-relative-to-fmt')``.

This is surprising outcome because we know that FMT intervention was supposed to rectify the decreasing diversity following allo-HCT.


This may be because these metagenomic samples are a subsample of the 16S samples in which we originally saw this trend. 

Let's checkout the 16S amplicon verison of this visualization: 
(link to 16s viz)

We can also investigate Shannon's entropy using similar steps. However general trends between Shannons and Observed Features remain the same. 


```bash
qiime diversity alpha \
    --i-table autofmt-table-rf.qza \
    --p-metric "shannon" \
    --o-alpha-diversity shannon-autofmt-bracken-rf
```
```bash
qiime longitudinal linear-mixed-effects \
  --m-metadata-file ../new-sample-metadata.tsv shannon-autofmt-bracken-rf.qza \
  --p-state-column day-relative-to-fmt \
  --p-individual-id-column PatientID \
  --p-metric shannon_entropy \
  --o-visualization lme-shannon-features-FMT.qzv
```

```bash
qiime longitudinal linear-mixed-effects \
  --m-metadata-file ../new-sample-metadata.tsv shannon-autofmt-bracken-rf.qza \
  --p-state-column day-relative-to-fmt \
  --p-individual-id-column PatientID \
  --p-metric shannon_entropy \
  --o-visualization lme-shannon-features-FMT.qzv
```
Now that we better understand community richness trends, lets look at differences in microbial composition.

Let investigate this by looking at Bray Curtis: 
```bash
qiime diversity beta \
  --i-table autofmt-table-rf.qza \
  --p-metric braycurtis \
  --o-distance-matrix braycurtis-autofmt
```
Now that we have our Bray Curtis distance matrix, lets visualize this using a PCOA plot.

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
Another way we can look at microbial composition is to investigate the taxa barplot. One thing to Note, these tend to be even more chaotic then the Amplicon data. 
```bash
qiime taxa barplot \
  --i-table table-bracken.qza \
  --i-taxonomy taxonomy-bracken.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --o-visualization taxa-bar-plot.qzv
```

Lets highlight some features of metagenomic data that we wouldn't see in Amplicon data.

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


```bash
qiime feature-table filter-samples \
  --i-table autofmt-table.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --p-where "[categorical-time-relative-to-fmt]='peri'" \
  --o-filtered-table peri-fmt-table
```

```bash
# 2 samples from subject FMT0009
echo SampleID > samples-to-keep.tsv
echo SRR14092317 > samples-to-keep.tsv
```
```bash
qiime feature-table filter-samples \
  --i-table peri-fmt-table.qza \
  --m-metadata-file samples-to-keep.tsv \
  --o-filtered-table id-filtered-peri-fmt-table.qza
```
```bash
qiime feature-table summarize \
  --i-table id-filtered-peri-fmt-table.qza \
  --m-sample-metadata-file ../new-sample-metadata.tsv \
  --o-visualization id-filtered-peri-fmt-table.qzv
```
```bash
 qiime composition ancombc \
  --i-table id-filtered-peri-fmt-table.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --p-formula autoFmtGroup \
  --o-differentials differentials-peri-autofmt.qza
```
```bash
  qiime longitudinal feature-volatility \
  --i-table  autofmt-table-p8.qza \
  --m-metadata-file ../new-sample-metadata.tsv 	pcoa-braycurtis-auto-fmt.qza obs-table-bracken-rf.qza \
  --p-state-column week-relative-to-hct \
  --p-individual-id-column PatientID \
  --output-dir longitudinal-feature-volatility-hct
```




