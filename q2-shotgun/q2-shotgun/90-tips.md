# Tips for shotgun metagenome analysis with QIIME 2

## Use temporary directories with a lot of storage capacity

Use a temporary directory where you have a lot of storage capacity. On a slurm-based system, set the temporary directory in your job script, so the environment is set up correctly where the job is running. I include the following two lines in all of my job scripts (the second line isn't required, but allows me to confirm that it is set correctly on the node where the job is running).

```shell
export TMPDIR=/scratch/jgc53/temp
echo $TMPDIR
```

## Use the QIIME 2 artifact cache for reference data

Use the [QIIME 2 artifact cache](https://caporasolab.us/developing-with-qiime2/40-reference/10-api/20-cache/00-index.html) to avoid unzipping large artifacts frequently.

