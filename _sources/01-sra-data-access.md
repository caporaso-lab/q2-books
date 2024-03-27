# Data access from SRA and other resources

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

### Required databases

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

## Data retrieval from SRA

The data we are using in this tutorial can be fetched from the SRA repository using the [q2-fondue](https://github.com/bokulich-lab/q2-fondue) plugin. You can fetch the list of accession IDs using the following command:

```shell
wget TODO
```

Next, we need to import the accession IDs into a QIIME 2 artifact:

```shell
qiime tools import \
  --type NCBIAccessionIDs \
  --input-path ids.tsv \
  --output-path ./moshpit_tutorial/ids.qza
```

Finally, we can use the `get-sequences` action to download the data (please insert your e-mail address in the `--p-email` parameter):

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
This will download all the sequences into the QIIME 2 cache. It is a lot of data, so keep in mind that depending on your network speed, this might take a while. Once the data is downloaded, you can proceed to one (or more) of the following steps:

- Annotation of reads (TODO)
- Generation and annotation of contigs (TODO)
- Generation and annotation of MAGs (TODO)

Before we jump into the next sections, lets discuss our parsl_config! We are using Parsl to parallelize our computationally expensive jobs. This will be used to run `Kraken` and `Eggnog-Mapper`. Here we can decide how many workers, cores, nodes and blocks.

