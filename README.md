# 1000Genomes Workflow

The 1000 genomes project provides a reference for human variation, having reconstructed the genomes of 2,504 individuals across 26 different populations to energize these approaches. This workflow identifies mutational overlaps using data from the 1000 genomes project in order to provide a null distribution for rigorous statistical evaluation of potential disease-related mutations. The workflow fetchs, parses, and analyzes data from the [1000 genomes Project](https://www.internationalgenome.org) (see ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/release/). It cross-matches the extracted data (which person has which mutations), with the mutation's sift score (how bad it is). Then it performs a few analyses, including plotting.

The figure below shows a branch of the workflow for the analysis of a single chromossome.

<p align="center">
  <img src="docs/images/1000genome.png?raw=true" width="600">
</p>

_Individuals_. This task fetches and parses the Phase3 data from the 1000 genomes project by chromosome. These files list all of Single nucleotide polymorphisms (SNPs) variants in that chromosome and which individuals have each one. SNPs are the most common type of genetic variation among people, and are the ones we consider in this work. An individual task creates output files for each individual of _rs numbers_ 3, where individuals have mutations on both alleles.

_Populations_. The 1000 genome project has 26 different populations from many different locations around the globe. A population task downloads a file per population selected. This workflow uses five super populations: African (AFR), Mixed American (AMR), East Asian (EAS), European (EUR), and South Asian (SAS). The workflow also uses ALL population, which means that all individuals from the latest release are considered.

_Sifting_. A sifting task computes the SIFT scores of all of the SNPs variants, as computed by the Variant Effect Predictor (VEP). SIFT is a sequence homology-based tool that Sorts Intolerant From Tolerant amino acid substitutions, and predicts whether an amino acid substitution in a protein will have a phenotypic effect. For each chromosome the sifting task processes the corresponding VEP, and selects only the SNPs variants that has a SIFT score, recording in a file (per chromosome) the SIFT score and the SNPs variants ids, which are: (1) rs number, (2) ENSEMBL GEN ID, and (3) HGNC ID.

_Pair Overlap Mutations_. This task measures the overlap in mutations (also called SNPs variants) among pairs of individuals by population and by chromosome.

_Frequency Overlap Mutations_. This tasks measures the frequency of overlapping in mutations by selecting a number of random individuals, and selecting all SNPs variants without taking into account their SIFT scores.


This workflow is based on the application described in: https://github.com/rosafilgueira/Mutation_Sets

## Prerequisites

This workflow is fully compatible with Pegasus WMS, we have the following prerequisites:

- **Pegasus** - version 5.0 or higher
- **Python** - version 3.6 or higher
- **HTCondor** - version 9.0 or higher

Note that, there exists a version of this workflow comaptible with Pegasus 4.9 under the branch `4.9`.

## Running this worklow

Unzipping Input Data
---------------------
```
./prepare_input.sh
```

Creating a Workflow
---------------------
```
./daxgen.py
```
Or,
```
./daxgen.py -d 1000genome.dax -D 20130502 -f data.csv
```
This workflow assumes that all input data listed in the `data.csv` file is available in the `data/20130502` folder by default (but you can change that behavior with the `-D`).

Submitting a Workflow
---------------------

### HTCcondor

By default this workflow will run on a local available HTCondor pool, you have nothing to set.

#### Memory requirements

We discuss here some memory requirement for the *individuals* jobs which are by far the largest jobs of the workflow. This workflow processes a given number of chromosomes named `ALL.chrX.250000.vcf` where `X` is the number of the chromosome and `250000` is the number of lines of that file. If the workflow processes 10 chromosomes then we will have 10 *individuals* jobs and one *individuals_merge* job. However, because this file is extremely long (250k lines), we can create multiple individuals job to process one chromosome, then *individuals_merge* job will make sure we merge each chunk processed in parallel. For example if we create 5 *individuals* jobs per chromosome then eachjob will process only 50,000 lines instead of 250,000. If we have 10 chromosomes then we will have `10*5`  *individuals* jobs and `5`  *individuals_merge* jobs. 

| Total number of *individuals* job per chromosome | Input size per *individuals* job (number of lines) | Memory required per *individuals* job |
| :---                                             |    :----:                                          |                                  ---: |
| 2                                                | 125,000 / 250,000                                  | 6.10 GB                               |
| 5                                                | 50,000 / 250,000                                   | 3.93 GB                               |
| 10                                               | 25,000 / 250,000                                   | 3.17 GB                               |
| 16                                               | 15,625 / 250,000                                   | 2.93 GB                               |

#### Unsufficient memory for HTCondor slots

If HTCondor does not have enough memory to execute these jobs, Condor will send a SIGKILL (code -9) to the jobs. To avoid that you can configure HTCondor to use dynamic slot allocation (slots can grow on demand) with this (don't forget to restart Condor services after):

```
# dynamic slots
SLOT_TYPE_1 = cpus=100%,disk=100%,swap=100%
SLOT_TYPE_1_PARTITIONABLE = TRUE
NUM_SLOTS = 1
NUM_SLOTS_TYPE_1 = 1
```

### HPC clusters at NERSC
You can submit this workflow at The National Energy Research Scientific Computing Center (NERSC) on [Cori](https://docs.nersc.gov/systems/cori/) if you have an account there. You will havw to use Bosco to submit remotely.

#### Bosco

[Bosco](https://osg-bosco.github.io/docs/)

#### Submission mode

Then, when generating the workflow with `daxgen.py`, just set the flag `--execution-site cori` instead of the default `local` which will use an HTCondor pool.

#### Clustering mode
We have implement several clustering methods in that workflow, we cluster all the *individuals* and *individuals_merge* jobs together to improve performance.
By default there is no clustering used (for more information about clustering in Pegasus see [here](https://pegasus.isi.edu/documentation/user-guide/optimization.html?highlight=cluster#job-clustering)). 

However, interested users can use two different types of clustering that can improve performance:
 1. MPI: You can use [Pegasus MPI Cluster](https://pegasus.isi.edu/documentation/manpages/pegasus-mpi-cluster.html) mode  with the flag `--pmc`, which allow Pegasus to run multiple jobs inside using a classic leader and follower paradigm using MPI.
 2. MPI In-memory: You can also use an in-memory system called [Decaf](https://bitbucket.org/tpeterka1/decaf/) with the flag `--decaf` (_Warning_: these two options are mutually exclusive!) 

