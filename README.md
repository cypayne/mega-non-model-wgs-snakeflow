mega-non-model-wgs-snakeflow
================

## Quick install and run

If you would like to put this on your system and test it running on a
single node (more later about using SLURM for deployment across multiple
nodes) you have to clone this repository and then download the
pseudo-genome used for the included test data set (in `.test`).

You must have Snakemake in the active environment. I am currently
developing and testing this with snakemake 7.7.0.

In short, here are the steps to install and run the `.test`.

``` sh
# clone the repo
git clone git@github.com:eriqande/mega-non-model-wgs-snakeflow.git

# download the tarball with the genome in it and then move that
# into resources/
wget --load-cookies /tmp/cookies.txt "https://docs.google.com/uc?export=download&confirm=$(wget --quiet --save-cookies /tmp/cookies.txt --keep-session-cookies --no-check-certificate 'https://docs.google.com/uc?export=download&id=1LMK-DCkH1RKFAWTR2OKEJ_K9VOjJIZ1b' -O- | sed -rn 's/.*confirm=([0-9A-Za-z_]+).*/\1\n/p')&id=1LMK-DCkH1RKFAWTR2OKEJ_K9VOjJIZ1b" -O non-model-wgs-example-data.tar && rm -rf /tmp/cookies.txt

# untar the tarball
tar -xvf non-model-wgs-example-data.tar

# copy the genome from the extracted tarball into mega-non-model-wgs-snakeflow/resources/
cp non-model-wgs-example-data/resources/genome.fasta mega-non-model-wgs-snakeflow/resources/
```

Once that is set up, you can do a dry run like:

``` sh
conda activate snakemake
cd mega-non-model-wgs-snakeflow

# set the number of cores you have access to, to use in the
# following command.  Here I have 12.  You should set yours
# however is appropriate
CORES=12
snakemake --cores $CORES --use-conda --conda-frontend mamba -np
```

If that gives you a reasonable looking output (165 total jobs, lots of
conda environments to be installed, etc.) then take the `-np` off the
end of the command to actually run it:

``` sh
snakemake --cores $CORES --use-conda --conda-frontend mamba
```

Installing all the conda packages could take a while (2–30 minutes,
depending on your system). Once that was done, running all the steps in
the workflow on this small data set required less than 4 minutes on 12
cores of a single node from UC Boulder’s SUMMIT supercomputer.

## Condensed DAG for the workflow

Here is a DAG for the workflow on the test data in `.test`, condensed
into an easier-to-look-at picture by the `condense_dag()` function in
Eric’s [SnakemakeDagR](https://github.com/eriqande/SnakemakeDagR)
package. ![](README_files/test_run_dag_condensed.svg)<!-- -->

## What the user must do and values to be set, etc

-   `units.tsv`
-   Choose an Illuminaclip adapter fasta (in config)

## Assumptions

-   Paired end

## Things fixed or added relative to JK’s snakemake workflow

-   fastqc on both reads
-   don’t bother with single end
-   add adapters so illumina clip can work
-   benchmark each rule
-   use genomicsDBimport
-   allow for merging of lots of small scaffolds into genomicsDB

## Things that will be added in the future

-   Develop a sane way to iteratively bootstrap some base-quality score
    recalibration.

## Stepwise addition of new samples to the Workflow (and the Genomics Data bases)

I have made a scheme were we can start with one units.tsv file that
maybe only has six samples in it, and you can run that to completion.
Then you can update the units.tsv file to have two additional samples in
it, and that should then properly update the genomics data bases. This
is done by a system of writing Genomics\_DBI receipts that tell us what
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
give those newly-resequenced samples new sample\_id’s (and sample
numbers). That would work.)
