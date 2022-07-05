mega-non-model-wgs-snakeflow
================

-   [Major Update/Upgrade Notes](#major-updateupgrade-notes)
-   [Quick install and run](#quick-install-and-run)
    -   [So, what just happened there](#so-what-just-happened-there)
-   [Condensed DAG for the workflow](#condensed-dag-for-the-workflow)
-   [Running this with SLURM](#running-this-with-slurm)
-   [What the user must do and values to be set,
    etc](#what-the-user-must-do-and-values-to-be-set-etc)
    -   [`units.tsv`](#unitstsv)
    -   [`chromosomes.tsv`](#chromosomestsv)
    -   [`scaffold_groups.tsv`](#scaffold_groupstsv)
    -   [`config.yaml`](#configyaml)
-   [Bootstrapped Base Quality Score
    Recalibration](#bootstrapped-base-quality-score-recalibration)
    -   [What values should be chosen?](#what-values-should-be-chosen)
-   [Offloading results to google
    drive](#offloading-results-to-google-drive)
    -   [Google Drive Directory
        Structure](#google-drive-directory-structure)
-   [Assumptions](#assumptions)
-   [Things fixed or added relative to JK’s snakemake
    workflow](#things-fixed-or-added-relative-to-jks-snakemake-workflow)
-   [Eric’s list of future
    enhancements](#erics-list-of-future-enhancements)
-   [Stepwise addition of new samples to the Workflow (and the Genomics
    Data
    bases)](#stepwise-addition-of-new-samples-to-the-workflow-and-the-genomics-data-bases)

## Major Update/Upgrade Notes

-   I have now pulled the `bqsr-directory-sructure` branch into main.
    This lets us easily do one or more rounds of “bootstrapped base
    quality score recalibration.” The directory structure has been
    modified rather simply. For each round of BQSR (0 = no BQSR; 1 =
    using variants from 0 to do a round of BQSR and call variants from
    the results, 2 = using the variants from 1 to do another round of
    BQSR and call variants from the result; etc.), the entire old
    directory structure (except for the `slurm_logs`) get placed into a
    trunk directory of `bqsr-round-x` where `x` is 0 or 1 or 2, etc.
-   There are a few files/steps that could be marked as temporary, but I
    haven’t done that yet, because, while developing, it can be useful
    to have some of those stages for re-running from that point.
-   The way I have done the directory structure, there doesn’t seem to
    be a good way to mark, for example, the gVCFs from the 0-round of
    BQSR be deleted after the 1-round has been done, without also
    triggering deletion of the 1-round’s GVCFs. So, for now, this has to
    be done by hand.
-   Some new things are required in the config file now. Basically the
    new additions looks like this:

``` yaml
bqsr_rounds: 2
bqsr_maf: 0.0225
bqsr_qual: 37
bqsr_qd: 15

# the following must be a list, even if it is just one element
maf_cutoffs: [0.01, 0.05]
```

## Quick install and run

If you would like to put this on your system and test it running on the
tiny test data set it comes with on a single node (or across multiple
nodes if you are on a SLURM cluster), these are the s you have to take.
We assume that you already have `git` installed.

1.  Install Snakemake if you don’t already have it. I have been
    developing and testing this mostly on snakemake-6, but am not
    testing and using it with snakemake-7.7.0. You must have a *full
    installation* of snakemake, and you must have `mamba`. To install
    snakemake so that this all works, follow the installation directions
    at
    <https://snakemake.readthedocs.io/en/stable/getting_started/installation.html>.

If you already have snakemake, but it it version 6 or earlier, you
should create a new snakemake environment with the latest version. Check
the installation page to see what the latest version is and then tag the
name of your snakemake environment with it. For example, if the latest
stable version is 7.7.0, you command would look like:

``` sh
conda activate base
mamba create -c conda-forge -c bioconda -n snakemake-7.7.0 snakemake
```

2.  You must have cloned this repository. If you are savvy with this
    sort of thing, you might as well fork it then clone it. If not, the
    simplest way to clone it will be to use this command:

``` sh
git clone https://github.com/eriqande/mega-non-model-wgs-snakeflow.git
```

3.  When that is done, change into the repository directory, and
    activate the snakemake conda environment:

``` sh
cd mega-non-model-wgs-snakeflow/
conda activate snakemake-7.7.0
```

4.  The first thing we will do is a “dry-run” of the workflow. This
    tells you all the different steps that will be taken, but does not
    actually run them.

``` sh
 snakemake --cores 20 --use-conda  -np --configfile .test/config/config.yaml
```

-   The `--configfile` option tells snakemake to find all the
    configurations for the run in `.test/config/config.yaml`. This runs
    a very small test data set of 8 samples from fastq to VCF.
-   The `-np` option tells snakemake to do a dry run and also to print
    all the shell commands that it would use.

After you run that command, there should be a lot of output (one little
block for each job) and then a summary at the end that looks something
like this:

    Job stats:
    job                                   count    min threads    max threads
    ----------------------------------  -------  -------------  -------------
    all                                       1              1              1
    apply_bqsr                               16              1              1
    bcf_concat                                1              1              1
    bcf_concat_mafs                           5              1              1
    bcf_maf_section_summaries                30              1              1
    bcf_section_summaries                    18              1              1
    bung_filtered_vcfs_back_together         18              1              1
    combine_bcftools_stats                    3              1              1
    combine_maf_bcftools_stats                5              1              1
    concat_gvcf_sections                     24              1              1
    condense_variants_for_bqsr                2              1              1
    fastqc_read1                             22              1              1
    fastqc_read2                             22              1              1
    genomics_db2vcf                          18              2              2
    genomics_db_import_chromosomes           12              2              2
    genomics_db_import_scaffold_groups        6              2              2
    hard_filter_indels                       18              1              1
    hard_filter_snps                         18              1              1
    maf_filter                               30              1              1
    make_chromo_interval_lists               12              1              1
    make_gvcf_sections                      144              1              1
    make_indel_vcf                           18              1              1
    make_scaff_group_interval_lists           6              1              1
    make_snp_vcf                             18              1              1
    map_reads                                22              4              4
    mark_dp0_as_missing                      18              1              1
    mark_duplicates                           8              1              1
    multiqc                                   3              1              1
    recalibrate_bases                        16              1              1
    samtools_stats                           24              1              1
    trim_reads_pe                            22              1              1
    total                                   580              1              4

    This was a dry-run (flag -n). The order of jobs does not reflect the order of execution.

5.  Do the run. But only install the necessary software environments:

``` sh
snakemake --cores 20 --use-conda  --conda-create-envs-only --configfile .test/config/config.yaml
```

This can take 5 or 10 minutes, or even longer, but you only have to do
it once. After that, when you run the workflow, all the software
environments will already be in place.

6.  Once that has finished. Do a whole run of the test data set. Note
    that this is set up to use 20 cores, which is reasonable if you have
    checked out an entire node on SEDNA, using, for example
    `srun -c 20 --pty /bin/bash`. At any rate, to do the run you give
    this command:

``` sh
 snakemake --cores 20 --use-conda  --keep-going --configfile .test/config/config.yaml
```

When you do that it should take about 5 minutes to run through the whole
workflow on the tiny test data. Note that the multiqc step will fail.
This is a quirk of the tiny test data. I have never had a problem with
it failing on real, full-sized data sets. You will get an error message
about the multiqc rule failing, but, with the `--keep-going` option,
everything else will finish.

7.  Once that has finished. Do a dry run of snakemake again and you
    should see that all that remains is that pesky multiqc run (again,
    this won’t be a problem on real data) which is part of rule `all`:

``` sh
snakemake --cores 20 --use-conda  --keep-going  -np --configfile .test/config/config.yaml
```

### So, what just happened there

The upshot is that this workflow started with the fastq files in `.test`
that represent 8 samples sequenced across multiple lanes and prepared in
different libraries:

``` sh
(snakemake-7.7.0) [node08: mega-non-model-wgs-snakeflow]--% ls .test/data/fastq/
T199967_T2087_HY75HDSX2_L001_R1_001.fastq.gz  T199970_T2094_HY75HDSX2_L004_R2_001.fastq.gz  T199973_T2087_HY75HDSX2_L002_R1_001.fastq.gz
T199967_T2087_HY75HDSX2_L001_R2_001.fastq.gz  T199971_T2087_HY75HDSX2_L002_R1_001.fastq.gz  T199973_T2087_HY75HDSX2_L002_R2_001.fastq.gz
T199968_T2087_HY75HDSX2_L001_R1_001.fastq.gz  T199971_T2087_HY75HDSX2_L002_R2_001.fastq.gz  T199973_T2087_HY75HDSX2_L003_R1_001.fastq.gz
T199968_T2087_HY75HDSX2_L001_R2_001.fastq.gz  T199971_T2087_HY75HDSX2_L004_R1_001.fastq.gz  T199973_T2087_HY75HDSX2_L003_R2_001.fastq.gz
T199968_T2087_HY75HDSX2_L002_R1_001.fastq.gz  T199971_T2087_HY75HDSX2_L004_R2_001.fastq.gz  T199973_T2094_HY75HDSX2_L002_R1_001.fastq.gz
T199968_T2087_HY75HDSX2_L002_R2_001.fastq.gz  T199971_T2099_HTYYCBBXX_L002_R1_001.fastq.gz  T199973_T2094_HY75HDSX2_L002_R2_001.fastq.gz
T199969_T2087_HTYYCBBXX_L002_R1_001.fastq.gz  T199971_T2099_HTYYCBBXX_L002_R2_001.fastq.gz  T199973_T2094_HY75HDSX2_L003_R1_001.fastq.gz
T199969_T2087_HTYYCBBXX_L002_R2_001.fastq.gz  T199972_T2087_HTYYCBBXX_L003_R1_001.fastq.gz  T199973_T2094_HY75HDSX2_L003_R2_001.fastq.gz
T199969_T2087_HY75HDSX2_L002_R1_001.fastq.gz  T199972_T2087_HTYYCBBXX_L003_R2_001.fastq.gz  T199974_T2087_HY75HDSX2_L001_R1_001.fastq.gz
T199969_T2087_HY75HDSX2_L002_R2_001.fastq.gz  T199972_T2087_HY75HDSX2_L001_R1_001.fastq.gz  T199974_T2087_HY75HDSX2_L001_R2_001.fastq.gz
T199969_T2087_HY75HDSX2_L003_R1_001.fastq.gz  T199972_T2087_HY75HDSX2_L001_R2_001.fastq.gz  T199974_T2094_HY75HDSX2_L001_R1_001.fastq.gz
T199969_T2087_HY75HDSX2_L003_R2_001.fastq.gz  T199972_T2094_HTYYCBBXX_L004_R1_001.fastq.gz  T199974_T2094_HY75HDSX2_L001_R2_001.fastq.gz
T199970_T2087_HY75HDSX2_L003_R1_001.fastq.gz  T199972_T2094_HTYYCBBXX_L004_R2_001.fastq.gz  T199974_T2099_HY75HDSX2_L001_R1_001.fastq.gz
T199970_T2087_HY75HDSX2_L003_R2_001.fastq.gz  T199972_T2094_HY75HDSX2_L002_R1_001.fastq.gz  T199974_T2099_HY75HDSX2_L001_R2_001.fastq.gz
T199970_T2094_HY75HDSX2_L004_R1_001.fastq.gz  T199972_T2094_HY75HDSX2_L002_R2_001.fastq.gz
```

And then it downloaded the genome for those samples, trimmed the
fastq’s, fastqc-ed them, mapped them to the genome, marked duplicates,
created gVCF files for each sample, imported those gVCF files to a
genomics data base, genotyped the samples from those genomic data bases,
marked sites with 0 read depth as missing, did best-practices GATK
hard-filtering on those genotypes, combined a lot of VCF files across
multiple regions of the genome into a single VCF file called
`results/vcf/all-filtered.vcf.gz`, and then printed out some summaries
of those variants using `bcftools stats`. You can have a look at that at
the VCF file with the command:

``` sh
zcat results/bqsr-round-{x}/bcf/all.bcf | less
```

Where {x} is 0, 1, or 2. If you are doing this on a Mac, then you can
use `gzcat` instead of `zcat`.

Additionally, the log files from every job that got run have been
recorded in various directories and files in
`results/bqsr-round-{x}/logs`.

Finally, run-time information (how long it took, how much memory was
required, how much disk I/O occurred) for every one of the jobs that ran
is recorded in various directories and files in
`results/bqsr-round-{x}/benchmarks`. This can be a treasure trove for
estimating how long different jobs/steps of this workflow will take on
new data sets.

All the files generated by the workflow are stored in

-   `resources`: downloaded and indexed genomes, etc. This also contains
    some adapter sequence files for trimmomatic that are distributed
    with this repo.
-   `results`: all the logs, all the outputs, etc.

A number of files are temporary files and are deleted after all
downstream products they depend on have been produced. There are many
more such files to mark as temporary, but I will do that after I have
used this updated workflow to finish out a long project.

Some files are marked as *protected* so that they cannot easily be
accidentally deleted or modified, such as the files in:

    results/bqsr-round-{x}/bcf

It would be typical practice to copy and archive all those to some other
place upon completion of the project.

The following section shows a nice acylic directed graph diagram of all
the steps in the workflow.

## Condensed DAG for the workflow

Here is a DAG for the workflow on the test data in `.test`, condensed
into an easier-to-look-at picture by the `condense_dag()` function in
Eric’s [SnakemakeDagR](https://github.com/eriqande/SnakemakeDagR)
package. ![](README_files/test_run_dag_condensed.svg)<!-- -->

## Running this with SLURM

This repository includes a snakemake profile that allows all the jobs in
the workflow to be dispatched via the SLURM scheduler. This can be
really handy. To test this on SEDNA, for example, do this:

1.  Remove the `results` and the genome parts in the `resources`
    directory, so that snakemake will run through the entire workflow,
    again:

``` sh
rm -rf resources/genome* results
```

The `-f` in the `-rf` option in the command above is used to override
the write-protection on some of the files.

2.  Do a dry run using the SEDNA slurm profile:

``` sh
snakemake --profile hpcc-profiles/slurm/sedna -np --configfile .test/config/config.yaml
```

You should get dry-run output like before.

3.  Make sure you have another shell available that you can put this
    command into, in order to see your SLURM job queue:

``` sh
squeue -u $(whoami) -o "%.12i %.9P %.50j %.10u %.2t %.15M %.6D %.18R %.5C %.12m"
```

4.  Start the snakemake job, using the slurm profile:

``` sh
snakemake --profile hpcc-profiles/slurm/sedna --configfile .test/config/config.yaml
```

While this is running, go to your other shell and use the above squeue
command to see all of your jobs that are queued or running. (To be
honest, there seems to be some latency with squeue on SEDNA. Since all
these jobs are super short, it might be that they are not there long
enough for squeue to show them. Instead, you can use
`sacct -u $(whoami)` to see all those jobs when running or completed).

## What the user must do and values to be set, etc

### `units.tsv`

The user has to make a file that lists all the different *units* of a
single sample. Typically different units are different fastq files that
hold sequences from a sinble biological sample. For example, the same
sample might have been sequenced on different lanes, or on different
machines, or it might have been prepared in more than a single library
prep. All that can be accounted for. The `units.tsv` file holds a lot of
necessary information for each sample. Here is a link to the `units.tsv`
file used for the `.test` data set:

<https://github.com/eriqande/mega-non-model-wgs-snakeflow/blob/main/.test/config/units.tsv>

All columns are required. Study it!

### `chromosomes.tsv`

The user must make this file that tells the workflow about the different
fully assembled chromosomes in the reference genome. Here is an example:

<https://github.com/eriqande/mega-non-model-wgs-snakeflow/blob/main/.test/config/chromosomes.tsv>

It is a TAB separated values file. You have to follow the format
exactly. The file can easily be made from the `.fai` file for the
genome. You can modify the helper script at
`workflow/prepare/make_chromosomes_and_scaffolds.R` to prepare this file
for yourself. Here is a link that that file if you want to see the
contents:

<https://github.com/eriqande/mega-non-model-wgs-snakeflow/blob/main/workflow/prepare/make_chromosomes_and_scaffolds.R>

The workflow operates a lot on individual chromosomes to allow
parallelization, so this is critical information.

### `scaffold_groups.tsv`

The user must make this file that tells snakemake which collections of
scaffolds should be merged together into scaffold groups.  
Here is what the `scaffold_groups.tsv` file looks like:

<https://github.com/eriqande/mega-non-model-wgs-snakeflow/blob/main/.test/config/scaffold_groups.tsv>

You have to follow the format, exactly. Also, the order of the scaffolds
in this file must match the order of the scaffolds in the reference
genome EXACTLY.

This file can also be made from the `.fai` file for the genome using the
helper script at `workflow/prepare/make_chromosomes_and_scaffolds.R`:

<https://github.com/eriqande/mega-non-model-wgs-snakeflow/blob/main/workflow/prepare/make_chromosomes_and_scaffolds.R>

### `config.yaml`

The user must make a `config.yaml` file. It serves a lot of purposes,
like:

-   giving the relative path to the `units.tsv`, `chromosomes.tsv`, and
    `scaffold_groups.tsv` files. (When we say “relative” path here we
    mean relative to the top level of the repo directory where the
    snakemake command will be given.)
-   Giving the URL from which the reference genome can be downloaded.
    (If there is not a URL for it, then just copy the reference FASTA
    file to `resources/genome.fasta`).
-   The location of the adapter file for Trimmomatic must be specified.
    The correct one to use depends on what sequencing platform your data
    come from.
-   The BQSR parameters as described in the next section.
-   The google drive directory to copy results back to, if desired. See
    below.
-   Some parameters can be set here; however, some of the YAML blocks
    here are vestigial and need to be cleaned up. Not all of these
    options actually change things. For now, ask Eric for help…

The current `config.yaml` file in the test directory can be viewed at:

<https://github.com/eriqande/mega-non-model-wgs-snakeflow/blob/main/.test/config/config.yaml>

As mentioned above, there is a little bit of cruft in it that should
stay in there for now, but which ought to be cleaned up, ultimately.

## Bootstrapped Base Quality Score Recalibration

The GATK folks insist that Base Quality Score Recalibration (BQSR) is a
very important step—even for non-model organisms that don’t have a
well-curated data base of known variants. Back in the days of the
Illumina Hi-Seq, I had been somewhat skeptical of the importance of BQSR
for non-model organisms. However, not that sequencing is mostly
happening on NovaSeq machines, my views have changed somewhat, because
these sequencing machines only deliver four possible base quality
scores, and one of those, `#` is given to uncalled bases. Observe:

``` r
# here we extract all the base quality scores recorded by the
# machine for the fastq file 
# .test/data/fastq/T199967_T2087_HY75HDSX2_L001_R1_001.fastq.gz
library(tidyverse)

bqs <- tibble(
  base_quals = read_lines(".test/data/fastq/T199967_T2087_HY75HDSX2_L001_R1_001.fastq.gz")
) %>%
  mutate(line = 1:n()) %>%
  filter(line %% 4 == 0) %>%
  mutate(singles = map(base_quals, strsplit, split = "")) %>%
  pull(singles) %>%
  unlist(.)

tibble(
  base_quals = bqs
) %>%
  count(base_quals) %>%
  mutate(
    ascii = map_int(base_quals, utf8ToInt),
    PHRED = ascii - 33
  )
```

    ## # A tibble: 4 × 4
    ##   base_quals       n ascii PHRED
    ##   <chr>        <int> <int> <dbl>
    ## 1 ,            68212    44    11
    ## 2 :            77998    58    25
    ## 3 #               17    35     2
    ## 4 F          1207941    70    37

So, these new machines produce a whole lot of sequence, but it really
only provides four possible values for the base qualities:

``` r
10 ^{-c(2, 11, 25, 37) / 10}
```

    ## [1] 0.6309573445 0.0794328235 0.0031622777 0.0001995262

Given this, some sort of empirical recalibration of the base quality
scores seems like it is probably a very good idea.

The problem with this for non-model organisms is that such species don’t
have a well-known data base of variants that can be used for the base
score recalibration. In that case, the GATK folks recommend using
“high-confidence” variants as the the know variant set, and then
possibly doing several rounds of this. Doing so requires a boatload of
computation, but it might be worthwhile, so this workflow has been set
up to do that.

Of course, it is hard to know what is meant by a “high-confidence”
variant. The purpose of having known variants is so that they don’t get
included in making an emprirical model of base quality scores.
Basically, GATK goes through the BAMs and it assumes that any base that
is not the reference base is an error, and it tallies those up, broken
down by a number of sequence and read-group features, and uses that to
estimate what the true sequencing error rate is. Of course, you don’t
reference mismatches at sites that are actually variants to contribute
to this calculation—because those mismatches are not actually errors.
But we are in a chicken and egg situation here—we don’t know which
variants are real and which are not!

I am of the mind that it is good to include as many known variants as
you can (because, otherwise, the BQSR model will tell us that there are
a lot of errors). But, on the other hand, you don’t want to include
dubious sites in that known variant set, because mismatches at those
sets are likely actually errors and should be counted.

It is important to recognize that the known variants are only excluded
from the BQSR model-building stage. Those sites themselves will still
get recalibrated once the model is made. And, if there isn’t a strong
pattern to the mismatches that occur because some true variants were
left out of the known variants data base, the consequences might not be
too bad.

All this is to say that this is a pretty inexact science. Nonetheless,
we have a few parameters that can be set in the config to select the
“known variants” set:

-   `bqsr_maf`. Sites with a minor allele frequency less than this will
    not be included in the known variants set. The idea is that you are
    more confident that a variant is real if it actually was observed in
    more than one or just a few individuals. Not only that, but, of all
    the true variants found in an individual, only somewhat less than
    `2 * bqsr_maf` will be discarded because of this filter.
    Accordingly, the default value in the test config is 0.0225, which
    means that about 3.5% of the mismatches in any actual variants in
    any *individual* will still be called mismatches. That seems OK to
    me.

-   `bqsr_qual`. Only sites with a variant quality (QUAL) score equal to
    or greater than this value will be retained in the known-variants
    set. For the test data set, this is set at 37, but should be larger
    (perhaps 100) for data sets with more depth and more individuals.

-   `bqsr_qd`. Only sites with a `QD`—a variant quality score,
    normalized by the number of reads—will be retained. `INFO/QD` is
    calculated by GATK. If it is low, it means that a variant has been
    called but the base quality scores for that variant are low on most,
    if not all, of the reads supporting that variant. The value in the
    .test data set is 15, but the effect of that should be investigated.

### What values should be chosen?

This is still an area where there are no solid answers; however, it can
be helpful to look at the distribution of the QUAL and the QD values.
The workflow is set up to make it easy to get those values and do all
the quality control steps in the workflow, and then you can investigate
the results to choose values of the three `bqsr_*` config parameters
listed above. That is done by setting `bqsr_rounds: 0` in the
config.yaml, and then choosing the `dest_qc_0` and the
`dest_bqsr_histos_0` rules as targets for the first run. This will do
all the mapping and qc and one round of variant calling and filtering
and then it will compute QUAL and QD histograms. For the test data set
on a local set of cores that looks like this:

``` sh
snakemake --cores 6  --use-conda  dest_bqsr_histos_0 dest_qc_0 --configfile .test/config/config.yaml
```

For determining the effect of the values of `bqsr_qual` and `bqsr_qd`,
after this has run you can investigate the files:

    results/bqsr-round-0/qc/bqsr_relevant_histograms/qd.tsv    results/bqsr-round-0/qc/bqsr_relevant_histograms/qual.tsv

Versions of those files have been stored in this repo so that we can see
some example R code of tallying them up:

``` r
quals <- read_tsv("README_files/qual.tsv", col_names = c("value", "n")) %>%
  arrange(value) %>%
  mutate(
    fract = n / sum(n),
    cumul = cumsum(fract)
  )
qds <- read_tsv("README_files/qd.tsv", col_names = c("value", "n")) %>%
  arrange(value) %>%
  mutate(
    fract = n / sum(n),
    cumul = cumsum(fract)
  )
```

Now, look at the cumulative distribution of `qd` values:

``` r
ggplot(qds, aes(x = value)) +
  geom_col(aes(y = fract)) +
  geom_point(aes(y = cumul)) +
  geom_line(aes(y = cumul)) +
  xlab("INFO/QD value")
```

![](README_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

And the distribution of `qual` values:

``` r
g <- ggplot(quals, aes(x = value)) +
  geom_col(aes(y = fract)) +
  geom_point(aes(y = cumul)) +
  geom_line(aes(y = cumul)) +
  xlab("QUAL value")

g
```

![](README_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

That is a little harder to see, so we can limit it to QUAL values less
than 100:

``` r
g +
  xlim(0, 100)
```

    ## Warning: Removed 126 rows containing missing values (position_stack).

    ## Warning: Removed 1 rows containing missing values (geom_col).

    ## Warning: Removed 126 rows containing missing values (geom_point).

    ## Warning: Removed 126 row(s) containing missing values (geom_path).

![](README_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

Obviously, with this very small test data set, it is hard to interpret
these results. But, you can sort of see why it seems like for this test
data set, the values of `bqsr_qd = 15` and `bqsr_qual = 37` might be
reasonable.

You really should check these values for your own data set and select
values for those parameters accordingly.

## Offloading results to google drive

Set the config parameter `rclone_base` to the directory on google drive
that you want created (if necessary) and everything copied to. For eric,
this is like:

``` yaml
rclone_base: "gdrive-rclone:Bioinformatic-Project-Archives/mega-flow-test"
```

for the test data set.

I have an rclone profile to copy results back to google drive. After you
have finished a run, you can run the snakemake on the rule
`send_to_gdrive` target, like this (for SEDNA):

``` sh
snakemake -np --profile  hpcc-profiles/slurm/sedna  send_to_gdrive --configfile config/config.yaml
```

And it will print out a command line that does:

1.  copies multiqc.html and bcftools stats to a `qc_summaries`
    directories
2.  tarballs up the `qc`, `benchmarks`, and `logs` directories in all
    bqsr-round-X directories
3.  uses rclone to copy:
    -   the bams and gvcfs (and their indexes) from the final round of
        BQSR,
    -   the BCF files (and their indexes) from the final and all all
        previous rounds of BQSR
    -   the `qc` and `benchmark` tarballs from the ry bqsr-round-X
        directory,
    -   the `qc_summaries` directory,
    -   the `resources` directory with the indexed genome
    -   the `bq_variants` and `bq_recal_tables`
    -   Finally, if there is a `data` directory present at the top level
        we assume that is what holds the original fastqs, and we copy
        all that back, too!

For example, on the test data set it prints this out:

``` sh
mkdir -p results/qc_summaries/bqsr-round-{0..2};  
for i in {0..2}; do 
  cp -r results/bqsr-round-$i/qc/{multiqc.html,bcftools_stats/*.txt} results/qc_summaries/bqsr-round-$i/; 
done;  
for i in {0..2}; do 
tar -cvf results/bqsr-round-$i/qc.tar results/bqsr-round-$i/qc; gzip results/bqsr-round-$i/qc.tar;  
done;  
for i in {0..2}; do 
tar -cvf results/bqsr-round-$i/benchmarks.tar results/bqsr-round-$i/benchmarks; gzip results/bqsr-round-$i/benchmarks.tar; 
done; 
for i in {0..2}; do 
tar -cvf results/bqsr-round-$i/logs.tar results/bqsr-round-$i/logs; gzip results/bqsr-round-$i/logs.tar; 
done; 

rclone copy --dry-run  --drive-stop-on-upload-limit . gdrive-rclone:Bioinformatic-Project-Archives/mega-flow-test   \
  --include='config/**'  \
  --include='results/qc_summaries/**'  \
  --include='results/bqsr-round-{0,1,2}/{qc,benchmarks,logs}.tar.gz'  \
  --include='results/bqsr-round-{0,1,2}/{bcf,bq_recal_tables,bq_variants}/**'  \
  --include='resources/**'  --include='data/**'  --include='results/bqsr-round-2/gvcf/*' \
  --include='results/bqsr-round-2/recal/*'
```

But, it prints out out all in one line. You then need to copy that line
into the Unix terminal so you can execute it and provide rclone with
your password.

### Google Drive Directory Structure

The resulting directory structure on google drive then looks like
this—an example from a case where one round of BQSR was done:

``` sh
└── Archive-Directory  # this will be named different things for different runs
    ├── config # sample meta data and info for doing the run
    ├── data   # The fastqs
    │   ├── NVS144B_R1_Columbus
    │   └── NVS144B_R2_Columbus
    ├── resources # the indexed genome it was mapped it
    │   └── adapters # adapter seqs used for Trimmomatic
    └── results
        ├── bqsr-round-0
        │   └── bcf. # the BCF files with no BQSR. pass = passes filters, maf is minor allele freq
        ├── bqsr-round-1
        │   ├── bcf  # more BCF files.  These after a round of BQSR.
        │   ├── bq_recal_tables # BQSR info for each individual 
        │   ├── bq_variants. # the variants used as "known" for BQSR round 1
        │   ├── gvcf. # the gVCF files after this round of BQSR
        │   └── recal  # the bam files after this round of  BQSR
        └── qc_summaries. # multiqc summaries and bcftools stats for variants for each round
            ├── bqsr-round-0
            └── bqsr-round-1
```

Within each of the `results/bqsr-round-X` directories you will also find
tarballs of all the log files and all the benchmark files.

## Assumptions

-   Paired end

## Things fixed or added relative to JK’s snakemake workflow

-   fastqc on both reads
-   don’t bother with single end
-   add adapters so illumina clip can work
-   benchmark each rule
-   use genomicsDBimport
-   allow for merging of lots of small scaffolds into genomicsDB

## Eric’s list of future enhancements

-   Parallelize the GenotypeGVCFs step via a scatter-gather whereby the
    user can define `chromo_scatter.tsv` and a `scaff_group_scatter.tsv`
    files that break things down into X Mb blocks, where X might be
    like 5. It may be harder for the scaffold groups, because some of
    the scatters will include multiple scaffolds. But, we could easily
    rock that all together in R. I would rather the user do it up front
    so that the workflow does not depend on checkpoints.
-   Develop a sane way to iteratively bootstrap some base-quality score
    recalibration.

## Stepwise addition of new samples to the Workflow (and the Genomics Data bases)

I have made a scheme were we can start with one units.tsv file that
maybe only has six samples in it, and you can run that to completion.
Then you can update the units.tsv file to have two additional samples in
it, and that should then properly update the genomics data bases. This
is done by a system of writing Genomics_DBI receipts that tell us what
is already in there.

Here is how you can run it and test that system is working properly on
the small included test data set. First we run it on the first six
samples, using `.test/config/units-only-s001-s006.tsv` as the units
file. This file can be viewed
[here](https://github.com/eriqande/mega-non-model-wgs-snakeflow/blob/main/.test/config/units-only-s001-s006.tsv)

``` sh
# run the pipeline on the first six samples:
snakemake --use-conda --cores 6  --keep-going --config units=.test/config/units-only-s001-s006.tsv
```

That should run through just fine. In the above I set it to use 6 cores,
and, after all the conda environments have been installed, it takes
about 5 minutes to run through this small test data set on my old
(mid-2014) Mac laptop that has 8 cores.

After that has completed, you should have a look at all the ouputs in
results. The chromosome- and scaffold-group-specific VCFs are in
`results/vcf_sections`. Note that they haven’t been filtered at all.

Also, you can check the multiqc report by opening
`results/qc/multiqc.html`

Now. Let us pretend that we did that initial run with our first six
samples when those were the only samples we had. But now we want to add
two more samples: `s007` and `s008`. If we have kept the genomics
databases, they can simply be updated. The snakemake workflow does all
that for us. All we have to do is provide an updated units file that has
all our original 6 samples, just like before, but has a few more rows
for the units corresponding to `s007` and `s008`. Such a file can be
seen
[here](https://github.com/eriqande/mega-non-model-wgs-snakeflow/blob/main/.test/config/units.tsv).

We run that as shown below. We have to be careful to force re-running
two rules,

``` sh
# To add the final two samples
# re-run it with the standard config that
# has the units.tsv file with all 8 samples, and we will --forcerun the
# genomics_db2vcf rule, to make sure it notices that it needs to add
# some more samples to the genomics data bases. # We also --forcerun the
# multiqc step, since that has to do re-done with all the new samples.
snakemake --use-conda --cores 6  --keep-going \
   --config units=.test/config/units.tsv \
   --forcerun genomics_db2vcf multiqc 
```

**NOTE:** on this tiny data set in `./test`, everything works on this
*except* that multiqc fails, likely because there isn’t enough data for
one of the samples or something weird like that…At any rate, don’t be
alarmed by that failure. It doesn’t seem to happen on more complete data
sets.

**HUGE CRUCIAL NOTE:** You *cannot* use this process to add any
additional units of samples you have already run through the workflow.
If you do that, it will completely screw everything up. This is useful
only when you are adding completely new samples to the workflow, it is
not designed for adding more reads from any sample that has already been
put into the genomics data bases. (That said, if you got new sequences
on a new machine/flow-cell or library from a sample that you had already
run through the pipeline, and you wanted to compare the results from the
new sequences to those from the original sequences, you could simply
give those newly-resequenced samples new sample_id’s (and sample
numbers). That would work.)
