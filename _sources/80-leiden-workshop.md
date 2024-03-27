# Metagenomics with QIIME 2 - Leiden 2024
#Data access from SRA and other resources

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

