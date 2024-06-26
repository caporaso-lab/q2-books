# Tips for metagenome analysis with QIIME 2
```{warning}
Metagenomics analysis with QIIME 2 is in alpha release.
This means that results you generate should be considered preliminary, and NOT PUBLICATION QUALITY.
Additionally, interfaces are subject to change, and those changes may be backward incompatible (meaning that a command or file that works in one version of the QIIME 2 Shotgun Metagenomics distribution may not work in the next version of that distribution).
```

## Use temporary directories with a lot of storage capacity

Use a temporary directory where you have a lot of storage capacity. On a slurm-based system, set the temporary directory in your job script, so the environment is set up correctly where the job is running. I include the following two lines in all of my job scripts (the second line isn't required, but allows me to confirm that it is set correctly on the node where the job is running).

```shell
export TMPDIR=/scratch/jgc53/temp
echo $TMPDIR
```
(Slurm-ConFig) = 
## HPC Slurm Config Example for Eggnog mapper and Kraken
```
[parsl]

[[parsl.executors]]
class = "HighThroughputExecutor"
label = "default"
max_workers = 1

[parsl.executors.provider]
class = "SlurmProvider"
mem_per_node = 200
exclusive = false
walltime = "48:00:00"
nodes_per_block = 1
cores_per_node = 40
min_blocks = 1
max_blocks = 10
```
## Use the QIIME 2 artifact cache for reference data

Use the [QIIME 2 artifact cache](https://caporasolab.us/developing-with-qiime2/40-reference/10-api/20-cache/00-index.html) to avoid unzipping large artifacts frequently.

