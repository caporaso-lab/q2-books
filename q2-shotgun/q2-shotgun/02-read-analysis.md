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

```bash
qiime longitudinal linear-mixed-effects \
  --m-metadata-file ../new-sample-metadata.tsv obs-autofmt-bracken-rf.qza \
  --p-state-column day-relative-to-fmt \
  --p-individual-id-column PatientID \
  --p-metric observed_features \
  --o-visualization lme-obs-features-FMT.qzv
```

```bash
qiime longitudinal linear-mixed-effects \
  --m-metadata-file ../new-sample-metadata.tsv obs-autofmt-bracken-rf.qza \
  --p-state-column day-relative-to-fmt \
  --p-group-columns autoFmtGroup \
  --p-individual-id-column PatientID \
  --p-metric observed_features \
  --o-visualization lme-obs-features-treatmentVScontrol.qzv
```

```bash
 qiime longitudinal linear-mixed-effects \
   --m-metadata-file ../new-sample-metadata.tsv obs-autofmt-bracken-rf.qza \
   --p-state-column DayRelativeToNearestHCT \
   --p-individual-id-column PatientID \
   --p-metric observed_features \
   --o-visualization lme-obs-features-HCT.qzv
```
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

```bash
qiime diversity beta \
  --i-table autofmt-table-rf.qza \
  --p-metric braycurtis \
  --o-distance-matrix braycurtis-autofmt
```

```bash
qiime diversity pcoa \
  --i-distance-matrix braycurtis-autofmt.qza \
  qii--o-pcoa pcoa-braycurtis-auto-fmt.qza 
```

```bash
qiime emperor plot \
  --i-pcoa pcoa-braycurtis-auto-fmt.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --o-visualization braycurtis-auto-fmt-emperor.qzv 
```

```bash
qiime emperor plot \
  --i-pcoa pcoa-braycurtis-auto-fmt.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --p-custom-axes week-relative-to-fmt \
  --o-visualization braycurtis-auto-fmt-emperor-custom.qzv 
```

```bash
qiime taxa filter-table \
  --i-table autofmt-filt-table.qza \
  --i-taxonomy taxonomy-bracken.qza \
  --p-include Viruses \
  --o-filtered-table virus-autofmt-table
```
```bash
qiime feature-table filter-samples \
  --i-table virus-autofmt-table.qza \
  --p-min-frequency 1240 \
  --o-filtered-table autofmt-virus-feature-contingency-filtered-table.qza
```

```bash
qiime taxa barplot \
  --i-table  autofmt-virus-feature-contingency-filtered-table.qza \
  --i-taxonomy taxonomy-bracken.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --o-visualization virus-taxa-bar-plot.qzv
```


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




