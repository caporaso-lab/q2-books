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
```bash
qiime diversity alpha \
    --i-table autofmt-table-rf.qza \
    --p-metric "observed_features" \
    --o-alpha-diversity obs-autofmt-bracken-rf
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
qiime emperor plot \
  --i-pcoa pcoa-braycurtis-auto-fmt.qza \
  --m-metadata-file ../new-sample-metadata.tsv \
  --o-visualization braycurtis-auto-fmt-emperor.qzv 
```
