# Running jobs on HPC scheduler

Objectives:
* Learn how to submit a job to the HPC scheduler.


## SI Computing Cluster: Hydra

PDF presentation about Hydra: [https://github.com/SmithsonianWorkshops/2019-03-05-metabarcoding/raw/master/hydra_workshop_3_5.pdf](https://github.com/SmithsonianWorkshops/2019-03-05-metabarcoding/raw/master/hydra_workshop_3_5.pdf).

This presentation covers the following topics:

* Why HPC?
* SI Computing Cluster statistics
* Hydra cluster architecture
* Best practices
* Tips

Another great resource for learning all about Hydra is the HPC Wiki, which can be found at [https://confluence.si.edu/display/HPC](https://confluence.si.edu/display/HPC).

## Logging In

```console
$ ssh {user}@hydra-login01.si.edu
Password:
```

## Creating a project directory

```console
$ cd /pool/genomics/{user}
```

```console
$ mkdir hydra_workshop
```

```console
$ cd hydra_workshop
```

Set up project directory structure.

```console
$ mkdir jobs
```

```console
$ mkdir logs
```

```console
$ mkdir -p data/raw
$ mkdir -p data/results
```

```console
$ tree .
.
├── data
│   ├── raw
│   └── results
├── jobs
└── logs

5 directories, 0 files
```

## Modules

Modules are a way to set your system paths and environmental variables for use of a particular program or pipeline.

You can view available modules with the command:

```console
$ module avail
```

This will output all modules installed on Hydra. A complete list can also be found at [https://www.cfa.harvard.edu/~sylvain/hpc/module-avail.html](https://www.cfa.harvard.edu/~sylvain/hpc/module-avail.html).

There are additional module commands to find out more about a particular module. Let's use the bioinformatics program *IQ-TREE* to test these out.

```console
$ module whatis bioinformatics/iqtree
bioinformatics/iqtree: System paths to run IQTREE 1.6.10
```

```console
$ module help bioinformatics/iqtree

----------- Module Specific Help for 'bioinformatics/iqtree/1.6.10' ---------------------------


Purpose
-------
This module file defines the system paths for IQTREE 1.6.1
The compiled binary that you can now call are:
iqtree
To run threaded use -nt flag, see documentation for more information.

Documentation
-------------
http://www.cibiv.at/software/iqtree/

<- Last updated: Mon Feb 25 10:36:42 EST 2019, VLG ->
```

```console
$ module show bioinformatics/raxml
-------------------------------------------------------------------
/share/apps/modulefiles/bioinformatics/iqtree/1.6.10:

module-whatis	 System paths to run IQTREE 1.6.10
prepend-path	 PATH /share/apps/bioinformatics/iqtree/1.6.10/bin
module		 load gcc/4.9.2
conflict	 bioinformatics/iqtree
conflict	 bioinformatics/iqtree/1.6.1
conflict	 bioinformatics/iqtree/1.3.9
conflict	 bioinformatics/iqtree/1.4.1
conflict	 bioinformatics/iqtree/1.5.2
conflict	 bioinformatics/iqtree/1.5.0
conflict	 bioinformatics/iqtree/1.6b3
-------------------------------------------------------------------

```

Ok, now let's actually load IQ-TREE.

Nothing happens, but let's run a quick command to show that we have IQ-TREE loaded properly.

```console
$ iqtree -h
IQ-TREE multicore version 1.6.10 for Linux 64-bit built Feb 19 2019
Developed by Bui Quang Minh, Nguyen Lam Tung, Olga Chernomor,
Heiko Schmidt, Dominik Schrempf, Michael Woodhams.

Usage: iqtree -s <alignment> [OPTIONS]

GENERAL OPTIONS:
  -? or -h             Print this help dialog
  -version             Display version number
  -s <alignment>       Input alignment in PHYLIP/FASTA/NEXUS/CLUSTAL/MSF format
...
```


## Submitting a job

We will now be creating and submitting an IQ-TREE job to the compute cluster.

First of all, let's copy a sample input .phy file into our `data/raw` directory.

```console
$ cp cp /data/genomics/data/primates.dna.phy data/raw
```

Now that the input file is in place, we'll need to generate a job submission file.

We will be using the Hydra QSub Generation Utility to create this file. Go to [https://hydra-4.si.edu/tools/QSubGen/](https://hydra-4.si.edu/tools/QSubGen/) and fill in the following values:

* *CPU time*: short
* *Memory*: 1 GB
* *Type of PE*: multi-thread
* *Number of CPUs*: 4
* *Select the job's shell*: sh
* *Select which modules to add*: bioinformatics/iqtree
* *Job specific commands*:
iqtree -s ../data/raw/primates.dna.phy -nt $NSLOTS -pre ../data/results/primates.dna

* *Job Name*: iqtree-{user}
* *Log File Name*: ../logs/iqtree
* *Err File Name*: [blank]
* *Change to CWD*: Y
* *Join output & error files*: Y
* *Send email notifications*: Y
* *Email*: [your email]

## Monitoring your job

```console
$ qstat
```

After job has completed:

```console
$ qacct -j {job ID}
```

## Interactive queue

```console
$ qrsh -pe mthread 2
```

```console
$ pwd
/home/{user}
```

```console
$ cd /pool/genomics/{}/hydra_workshop
```

## Transferring files

[https://cyberduck.io/](CyberDuck)





