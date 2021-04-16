# nf-core/bactmap: Usage

## :warning: Please read this documentation on the nf-core website: [https://nf-co.re/bactmap/usage](https://nf-co.re/bactmap/usage)

> _Documentation of pipeline parameters is generated automatically from the pipeline schema and can no longer be found in markdown files._

## Introduction

This pipeline maps short reads (usually Illumina) to a bacterial reference (usually of the same species and the closest high quality genome available). For this purpose you will require a set of paired read files (single read file version coming in future version) and a reference genome in fasta format. The read file pairs are specified in a sample sheet  

## Samplesheet input

You will need to create a samplesheet file with information about the samples in your experiment before running the pipeline. Use this parameter to specify its location. It has to be a comma-separated file with 4 columns, and a header row as shown in the examples below.

```bash
--input '[path to samplesheet file]'
```

### Multiple replicates

The `group` identifier is the same when you have multiple replicates from the same experimental group, just increment the `replicate` identifier appropriately. The first replicate value for any given experimental group must be 1. Below is an example for a single experimental group in triplicate:

```bash
group,replicate,fastq_1,fastq_2
control,1,AEG588A1_S1_L002_R1_001.fastq.gz,AEG588A1_S1_L002_R2_001.fastq.gz
control,2,AEG588A2_S2_L002_R1_001.fastq.gz,AEG588A2_S2_L002_R2_001.fastq.gz
control,3,AEG588A3_S3_L002_R1_001.fastq.gz,AEG588A3_S3_L002_R2_001.fastq.gz
```

### Multiple runs of the same library

The `group` and `replicate` identifiers are the same when you have re-sequenced the same sample more than once (e.g. to increase sequencing depth). The pipeline will concatenate the raw reads before alignment. Below is an example for two samples sequenced across multiple lanes:

```bash
group,replicate,fastq_1,fastq_2
control,1,AEG588A1_S1_L002_R1_001.fastq.gz,AEG588A1_S1_L002_R2_001.fastq.gz
control,1,AEG588A1_S1_L003_R1_001.fastq.gz,AEG588A1_S1_L003_R2_001.fastq.gz
treatment,1,AEG588A4_S4_L003_R1_001.fastq.gz,AEG588A4_S4_L003_R2_001.fastq.gz
treatment,1,AEG588A4_S4_L004_R1_001.fastq.gz,AEG588A4_S4_L004_R2_001.fastq.gz
```

### Full design

A final design file consisting of both single- and paired-end data may look something like the one below. This is for two experimental groups in triplicate, where the last replicate of the `treatment` group has been sequenced twice.

```bash
group,replicate,fastq_1,fastq_2
control,1,AEG588A1_S1_L002_R1_001.fastq.gz,AEG588A1_S1_L002_R2_001.fastq.gz
control,2,AEG588A2_S2_L002_R1_001.fastq.gz,AEG588A2_S2_L002_R2_001.fastq.gz
control,3,AEG588A3_S3_L002_R1_001.fastq.gz,AEG588A3_S3_L002_R2_001.fastq.gz
treatment,1,AEG588A4_S4_L003_R1_001.fastq.gz,
treatment,2,AEG588A5_S5_L003_R1_001.fastq.gz,
treatment,3,AEG588A6_S6_L003_R1_001.fastq.gz,
treatment,3,AEG588A6_S6_L004_R1_001.fastq.gz,
```

| Column         | Description                                                                                                 |
|----------------|-------------------------------------------------------------------------------------------------------------|
| `group`        | Group identifier for sample. This will be identical for replicate samples from the same experimental group. |
| `replicate`    | Integer representing replicate number. Must start from `1..<number of replicates>`.                         |
| `fastq_1`      | Full path to FastQ file for read 1. File has to be zipped and have the extension ".fastq.gz" or ".fq.gz".   |
| `fastq_2`      | Full path to FastQ file for read 2. File has to be zipped and have the extension ".fastq.gz" or ".fq.gz".   |

An [example samplesheet](../assets/samplesheet.csv) has been provided with the pipeline.

## Running the pipeline

The typical command for running the pipeline is as follows:

```bash
nextflow run nf-core/bactmap --input samplesheet.csv -profile docker
```

This will launch the pipeline with the `docker` configuration profile. See below for more information about profiles.

Note that the pipeline will create the following files in your working directory:

```bash
work            # Directory containing the nextflow working files
results         # Finished results (configurable, see below)
.nextflow_log   # Log file from Nextflow
# Other nextflow hidden files, eg. history of pipeline runs and old logs.
```

### Optional parameters

1. Index the reference sequence using [`bwa index`](https://github.com/lh3/bwa)
2. (Optionally if `params.trim` is set) trim reads using [`fastp`](https://github.com/OpenGene/fastp). The default trimming parameters are

   ```console
   --cut_front --cut_tail --trim_poly_x --cut_mean_quality 30 --qualified_quality_phred 30 --unqualified_percent_limit 10 --length_required 50
   ```

3. (Optionally if `params.depth_cutoff` is set) downsample to the reads based on estimated genome size using [`mash sketch`](https://github.com/marbl/Mash) and the required depth of coverage as set by the `depth_cutoff` params using [`rasusa`](https://github.com/mbhall88/rasusa)
4. Map reads to the indexed reference genome using [`bwa mem`](https://github.com/lh3/bwa) to produce a bam file
5. Sort the bam file using [`samtools`](http://www.htslib.org/doc/samtools.html)
6. Call variants using [`bcftools mpileup`](http://samtools.github.io/bcftools/bcftools.html). A minimum base quality of 20 is used for pre-filtering and the following fields are included in the resulting VCF file `FORMAT/AD,FORMAT/ADF,FORMAT/ADR,FORMAT/DP,FORMAT/SP,INFO/AD,INFO/ADF,INFO/ADR`. A haploid and multiallelic model is assumed. These defaults can be overridden using the `modules.bcftools_mpileup` params. The defaults are:

    ```console
    args  = '--min-BQ 20 --annotate FORMAT/AD,FORMAT/ADF,FORMAT/ADR,FORMAT/DP,FORMAT/SP,INFO/AD,INFO/ADF,INFO/ADR'
    args2 = '--ploidy 1 --multiallelic-caller'
    ```

7. Variants are filtered using [`bcftools filter`](http://samtools.github.io/bcftools/bcftools.html). The defaults params in `modules.bcftools_filter` are:

    ```console
    args = '--soft-filter LowQual --exclude "%QUAL<25 || FORMAT/DP<10 || MAX(FORMAT/ADF)<2 || MAX(FORMAT/ADR)<2 || MAX(FORMAT/AD)/SUM(FORMAT/DP)<0.9 || MQ<30 || MQ0F>0.1" --output-type z'
    ```

    There will be a row in filtered VCF file for each position in the reference genome. The position will either have value in the FILTER column of either `PASS` or `LowQual`.
8. Use the filtered VCF to create a pseudogenome based on the reference genome using the [`vcf2pseudogenome.py script`](https://github.com/nf-core/bactmap/blob/dev/bin/vcf2pseudogenome.py). The base in a sample at a position where the VCF file row that has `PASS` in FILTER will be either ref or alt and the appropriate base will be encoded at that position. The base in a sample at a position where the VCF file row that has `LowQual` in FILTER is uncertain will be encoded as a `N` character. Missing data will be encoded as a `-` character.
9. All samples pseudogenomes and the original reference sequence will concatenated together to produce a flush alignment where all the sequence for all samples at all positions in the original reference sequence will be one of `{G,A,T,C,N,-}`. This alignment can be used for other downstream processes such as phylogeny generation.
10. Optionally this alignment can be processed to produce a phylogeny as part of this pipeline. The number of constant sites in the alignment will be determined using [`snp-sites`](https://github.com/sanger-pathogens/snp-sites).
11. (Optionally if `params.remove_recombination` is set) remove regions likely to have been acquired by horizontal transfer and recombination and therefore perturb the true phylogeny using [`gubbins`](https://sanger-pathogens.github.io/gubbins/). This should only be run on sets of samples that are closely related and not for example on a set of samples that have diversity spanning that of the entire species.
12. Depending on the params set, run 0 - 4 tree building algorithms.
    * `params.modules.rapidnj.build = true` Build a neighbour-joining pylogeny using [`rapidnj`](https://birc.au.dk/software/rapidnj)
    * `params.modules.fasttree.build = true` Build an approximately-maximum-likelihood phylogeny using [`FastTree`](http://www.microbesonline.org/fasttree)
    * `params.modules.iqtree.build = true` Build a maximum-likelihood phylogeny using [`IQ-TREE`](http://www.iqtree.org)
    * `params.modules.raxmlng.build = true` Build a maximum-likelihood phylogeny using [`RAxML Next Generation`](https://github.com/amkozlov/raxml-ng)

### Updating the pipeline

When you run the above command, Nextflow automatically pulls the pipeline code from GitHub and stores it as a cached version. When running the pipeline after this, it will always use the cached version if available - even if the pipeline has been updated since. To make sure that you're running the latest version of the pipeline, make sure that you regularly update the cached version of the pipeline:

```bash
nextflow pull nf-core/bactmap
```

### Reproducibility

It's a good idea to specify a pipeline version when running the pipeline on your data. This ensures that a specific version of the pipeline code and software are used when you run your pipeline. If you keep using the same tag, you'll be running the same version of the pipeline, even if there have been changes to the code since.

First, go to the [nf-core/bactmap releases page](https://github.com/nf-core/bactmap/releases) and find the latest version number - numeric only (eg. `1.3.1`). Then specify this when running the pipeline with `-r` (one hyphen) - eg. `-r 1.3.1`.

This version number will be logged in reports when you run the pipeline, so that you'll know what you used when you look back in the future.

## Core Nextflow arguments

> **NB:** These options are part of Nextflow and use a _single_ hyphen (pipeline parameters use a double-hyphen).

### `-profile`

Use this parameter to choose a configuration profile. Profiles can give configuration presets for different compute environments.

Several generic profiles are bundled with the pipeline which instruct the pipeline to use software packaged using different methods (Docker, Singularity, Podman, Conda) - see below.

> We highly recommend the use of Docker or Singularity containers for full pipeline reproducibility, however when this is not possible, Conda is also supported.

The pipeline also dynamically loads configurations from [https://github.com/nf-core/configs](https://github.com/nf-core/configs) when it runs, making multiple config profiles for various institutional clusters available at run time. For more information and to see if your system is available in these configs please see the [nf-core/configs documentation](https://github.com/nf-core/configs#documentation).

Note that multiple profiles can be loaded, for example: `-profile test,docker` - the order of arguments is important!
They are loaded in sequence, so later profiles can overwrite earlier profiles.

If `-profile` is not specified, the pipeline will run locally and expect all software to be installed and available on the `PATH`. This is _not_ recommended.

* `docker`
  * A generic configuration profile to be used with [Docker](https://docker.com/)
  * Pulls software from Docker Hub: [`nfcore/bactmap`](https://hub.docker.com/r/nfcore/bactmap/)
* `singularity`
  * A generic configuration profile to be used with [Singularity](https://sylabs.io/docs/)
  * Pulls software from Docker Hub: [`nfcore/bactmap`](https://hub.docker.com/r/nfcore/bactmap/)
* `podman`
  * A generic configuration profile to be used with [Podman](https://podman.io/)
  * Pulls software from Docker Hub: [`nfcore/bactmap`](https://hub.docker.com/r/nfcore/bactmap/)
* `conda`
  * Please only use Conda as a last resort i.e. when it's not possible to run the pipeline with Docker, Singularity or Podman.
  * A generic configuration profile to be used with [Conda](https://conda.io/docs/)
  * Pulls most software from [Bioconda](https://bioconda.github.io/)
* `test`
  * A profile with a complete configuration for automated testing
  * Includes links to test data so needs no other parameters

### `-resume`

Specify this when restarting a pipeline. Nextflow will used cached results from any pipeline steps where the inputs are the same, continuing from where it got to previously.

You can also supply a run name to resume a specific run: `-resume [run-name]`. Use the `nextflow log` command to show previous run names.

### `-c`

Specify the path to a specific config file (this is a core Nextflow command). See the [nf-core website documentation](https://nf-co.re/usage/configuration) for more information.

#### Custom resource requests

Each step in the pipeline has a default set of requirements for number of CPUs, memory and time. For most of the steps in the pipeline, if the job exits with an error code of `143` (exceeded requested resources) it will automatically resubmit with higher requests (2 x original, then 3 x original). If it still fails after three times then the pipeline is stopped.

Whilst these default requirements will hopefully work for most people with most data, you may find that you want to customise the compute resources that the pipeline requests. You can do this by creating a custom config file. For example, to give the workflow process `star` 32GB of memory, you could use the following config:

```nextflow
process {
  withName: star {
    memory = 32.GB
  }
}
```

See the main [Nextflow documentation](https://www.nextflow.io/docs/latest/config.html) for more information.

If you are likely to be running `nf-core` pipelines regularly it may be a good idea to request that your custom config file is uploaded to the `nf-core/configs` git repository. Before you do this please can you test that the config file works with your pipeline of choice using the `-c` parameter (see definition above). You can then create a pull request to the `nf-core/configs` repository with the addition of your config file, associated documentation file (see examples in [`nf-core/configs/docs`](https://github.com/nf-core/configs/tree/master/docs)), and amending [`nfcore_custom.config`](https://github.com/nf-core/configs/blob/master/nfcore_custom.config) to include your custom profile.

If you have any questions or issues please send us a message on [Slack](https://nf-co.re/join/slack) on the [`#configs` channel](https://nfcore.slack.com/channels/configs).

### Running in the background

Nextflow handles job submissions and supervises the running jobs. The Nextflow process must run until the pipeline is finished.

The Nextflow `-bg` flag launches Nextflow in the background, detached from your terminal so that the workflow does not stop if you log out of your session. The logs are saved to a file.

Alternatively, you can use `screen` / `tmux` or similar tool to create a detached session which you can log back into at a later time.
Some HPC setups also allow you to run nextflow within a cluster job submitted your job scheduler (from where it submits more jobs).

#### Nextflow memory requirements

In some cases, the Nextflow Java virtual machines can start to request a large amount of memory.
We recommend adding the following line to your environment to limit this (typically in `~/.bashrc` or `~./bash_profile`):

```bash
NXF_OPTS='-Xms1g -Xmx4g'
```
