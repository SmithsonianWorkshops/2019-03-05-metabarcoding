# QIIME 2 Tutorial

In this tutorial, we will run part of the [QIIME 2](https://docs.qiime2.org/2022.2/tutorials/moving-pictures/) Moving Pictures tutorial on Hydra, adjusted for the COI data generated recently at NMNH.

**To run QIIME2, you will need to either load the qiime2 module or, if you've installed qiime2 on your own using conda, load the conda module and activate the environment**

```
module load bio/qiime2/2021.11
```

or

```
module load tools/conda
start-conda
conda activate qiime2-2021.11
```

*Copy sample data to your space on Hydra*
1. Change directory to your space on Hydra (e.g. `cd /scratch/genomics/USER`)
2. Make a new directory for the data (e.g. `mkdir qiime2_tutorial`) and `cd` into that new directory.
3. Set up a directory structure for your `qiime2_tutorial` "project." Run `mkdir -p jobs logs data/raw data/raw/reads data/working data/results`.
    Your directory should look like this:

```
    .
    ├── data
    │   ├── raw
    │   │   └── reads
    │   ├── results
    │   └── working
    ├── jobs
    └── logs
```

5. We've downsampled the COI data from a wetlab workshop and put it here: `/data/genomics/workshops/qiime2/data`
6. Choose two pairs of sequences and use `cp` to copy the sequences into your `data/raw/reads` directory and the metadata (`sample-metadata.tsv`) into your `raw` directory.
7. Check to see that your data are there with `ls data/raw`. You should have two `fq.gz` files, R1 and R2 for two samples and a `.tsv`
8. Edit the metadata file to include only the rows for the samples you have chosen. (Hint: if you are editing the file with `nano`, use `ctrl-k` to cut whole lines of text).

### Basics on using QIIME commands

All the QIIME commands start with `qiime` followed by commands, sub-commands and then options.

You can list all of the commands with: `qiime --help`

And then help for a command (or called a plugin in some cases) with: `qiime tools --help`

And then the options for a subcommand with `qiime tools import --help`

A useful thing is that tab-completion works for these commands. If you type `qiime<space>` and then hit the tab key twice, all the commands are listed. If that doesn't work, you may need to enter this command to enable the tab-completions: `source tab-qiime`

### Import data into QIIME

All data that is used as input to QIIME 2 is in form of QIIME 2 artifacts, which contain information about the type of data and the source of the data. The file extension used for these is `.qza` which stands for "qiime zip artifact." The first thing we need to do is import these sequence data files into a QIIME 2 artifact.

**SAMPLE JOB FILE:**

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -q sThC.q
#$ -l mres=8G,h_data=8G,h_vmem=8G
#$ -l cpu_arch='haswell|skylake'
#$ -cwd
#$ -j y
#$ -N qiime_import
#$ -o ../logs/qiime_test.log
#
# ----------------Modules------------------------- #
module load tools/conda
start-conda
conda activate qiime2-2021.11
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
qiime tools import \
 --type 'SampleData[PairedEndSequencesWithQuality]' \
 --input-path ../data/raw/reads \
 --input-format CasavaOneEightSingleLanePerSampleDirFmt \
 --output-path ../data/working/paired-end.qza
#
echo = `date` job $JOB_NAME done
```

Feel free to change the name of the output `.qza` file in the job file to something more informative. You can also try running this job via an interactive job using `qrsh`

Note that all the files in the directory given in `--input-path` are imported into the `.qza` file.

Want to see what other input formats are allowed? Run `qiime tools import --show-importable-formats` for a list.

`CasavaOneEightSingleLanePerSampleDirFmt` is the fastq format file produced by illumina sequencers (Casava is the name of the Illumina base caller). There's more information about importing files in the [qiime importing tutorial](https://docs.qiime2.org/2022.2/tutorials/importing/).

Hint: `.qza` files are acutally zipped archives. If you're curious what's contained in one, use `unzip paired-end.qza` to see the contents.

You can also use the command `qiime tools peak` to get a bit of information about a `.qza` file:

```
qiime tools peek ../data/working/paired-end.qza
UUID:        fde1b1c7-35f8-4021-804c-c4757fcb6887
Type:        SampleData[PairedEndSequencesWithQuality]
Data format: SingleLanePerSamplePairedEndFastqDirFmt
```

The command `qiime tools export` will extract the data portion of a `.qza` file.

```
qiime tools export --input-path ../data/working/paired-end.qza --output-path ../data/working
Exported paired-end.qza as SingleLanePerSamplePairedEndFastqDirFmt to directory .
```

That created an extra copy of the input files (see them with `ls ../data/working`), so we'll delete them with `rm ../data/working/*fastq.gz`.

Now we'll run a summarize command to look at the quality scores. Either run this within a job file with `qsub` or on an interactive job with `qrsh`

```
qiime demux summarize \
  --i-data ../data/working/paired-end.qza \
  --o-visualization ../data/results/paired-end.qzv
```

The file extension use for visualizations is `.qzv`, which stands for "qiime zip visualization." They can contain tables, images or other data. These are also zip archives if you want to try `unzip` on them.

View paired-end.qzv here: https://view.qiime2.org/ and have a look at the quality of your sequences. You can also download a copy from [here](https://github.com/SmithsonianWorkshops/2019-03-05-metabarcoding/raw/master/Metabarcoding_with_Qiime/qiime_artifacts/paired-end.qzv)

### Trim primers
Now we'll trim the primers off the sequences with cutadapt, which is available through QIIME2. Run this with a job file and `qsub`

```
qiime cutadapt trim-paired \
 --i-demultiplexed-sequences ../data/working/paired-end.qza \
 --p-cores $NSLOTS \
 --p-front-f GGWACWGGWTGAACWGTWTAYCCYCC \
 --p-front-r TANACYTCNGGRTGNCCRAARAAYCA \
 --o-trimmed-sequences ../data/working/paired-end-primers-trimmed.qza
```

`--p-cores` is the number of CPUs to use. By using `$NSLOTS`, the cluster software will use the value of CPUs requested for your job.

Now we'll run a summarize command to look at the quality scores. Either run this within a job file with `qsub` or on an interactive job with `qrsh`

```
qiime demux summarize \
  --i-data ../data/working/paired-end-primers-trimmed.qza \
  --o-visualization ../data/results/paired-end-primers-trimmed.qzv
```

View demux.qzv here: https://view.qiime2.org/ and have a look at the quality of your sequences. From these plots, we'll decide how much to trim the sequences for quality. A sample files is available [here](https://github.com/SmithsonianWorkshops/2019-03-05-metabarcoding/raw/master/Metabarcoding_with_Qiime/qiime_artifacts/paired-end-primers-trimmed.qzv)

### DADA2 quality control

A lot of the magic of this pipeline happens in [DADA2](https://benjjneb.github.io/dada2/) (the tutorial on that site is also very good). The program does several things: trim low quality areas of sequence, merge forward & reverse reads, detect chimeras (described below), and denoise the sequence (also described below).

- Sequence trimming: we give at what base the forward and reverse reads should be trimmed. This is base don the visualization from the `qiime demux summarize` command.
- Merge reads: For the region of the COI locus being sequenced forward and reverse reads overlap to give the full length of the sequenced region. The program takes the two reads and merges them into one full-length sequence of the locus.
- Chimera detection: Chimeras are reads that are determined to contained sequence from two disparate pieces of DNA. This can be due to PCR or sequencing error.
- Denoising: With amplicon sequencing you expect to sequence many copies of each PCR product. Due to sequencing (or PCR) error there can be small variations in the sequence. Denoising takes the sequence information and what is known about Illumina sequencing errors to detect and fix errors in a read.

Let's run dad2, this will take the longest of anything we've done so far, so use `qsub` and a job file for this command.

```
qiime dada2 denoise-paired \
 --i-demultiplexed-seqs ../data/working/paired-end-primers-trimmed.qza \
 --p-n-threads $NSLOTS \
 --p-trunc-len-f 270 \
 --p-trunc-len-r 170 \
 --o-representative-sequences ../data/working/rep-seqs-dada2.qza \
 --o-table ../data/working/table-dada2.qza \
 --o-denoising-stats ../data/working/stats-dada2.qza
```

- `--o-representative-sequences`: Sequence of each ASV
- `--o-table`: BIOM feature table (abundance of each sequence for each sample)
- `--o-denoising-stats`: Denoising stats

While it's running, something to discuss is the change in metabarcoding from OTUs (Operational Taxonomic Units) to ASVs (Amplicon Sequence Variants). Prior to denoising, the common way to handle sequencing and PCR errors was to take each sequence and cluster it using a threshold level of similarity. Those cluster became OTUs that were used in downstream analyses. With using denoising to limit errors, there was a move to keeping all (post-denoising) sequence variation, even single base differences (same as 100% OTUs in a clustering pipeline). These are ASVs which we'll be using in this analysis. The switch to ASVs was shown to lead to fewer errors (citation needed). Another advantage is that ASVs from different analyses could be combined for things like taxonomic identification which is not possible with OTUs because the results of a clustering are highly dependent on the input sequences.

### View dada2 results summary
Now we'll run a summarize command to look at how dada2 filtered our reads.

```
qiime metadata tabulate \
  --m-input-file ../data/working/stats-dada2.qza \
  --o-visualization ../data/results/stats-dada2.qzv
```

Then you will need to scp `stats-data2.qzv` to your computer so that you can view the results at https://view.qiime2.org/. You can download a sample [here](https://github.com/SmithsonianWorkshops/2019-03-05-metabarcoding/raw/master/Metabarcoding_with_Qiime/qiime_artifacts/stats-dada2.qzv)

Columns:
- `input`: starting number of (paired) reads.
- `filtered`: reads that pass quality filtering.
- `denoised`: reads after denoising.
- `merged`: reads where the forward and reverse reads were successfuly merged with overlap.
- `non-chimeric`: Merged reads that passed chimeric filters. These are the total number of usable sequences from the sample.

### Copy full dada2 results to data/working

You've been working on only one part of sequences. For the next section we're going to use all the sequences. You can copy the completed dada2 results (for the entire dataset) to your own `data/working` directory. This command will work if you are in `/scratch/genomics/USER/qiime2_tutorial`.

```
cp /data/genomics/workshops/qiime2/rep-seqs-dada2.qza /data/genomics/workshops/qiime2/table-dada2.qza /data/genomics/workshops/qiime2/stats-dada2.qza /data/genomics/workshops/qiime2/data/sample-metadata.tsv data/raw
```

Now we'll run a summarize command to look at how dada2 filtered our reads.

```
qiime metadata tabulate \
  --m-input-file ../data/working/stats-dada2.qza \
  --o-visualization ../data/results/stats-dada2.qzv
```

### Generate FeatureTable and FeatureData Summaries
The feature-table summarize command will give you information on how many sequences are associated with each sample and with each feature, histograms of those distributions, and some related summary statistics. The feature-table tabulate-seqs command will provide a mapping of feature IDs to sequences, and provide links to easily BLAST each sequence against the NCBI nt database.

```
qiime feature-table summarize \
  --i-table ../data/working/table-dada2.qza \
  --m-sample-metadata-file ../data/raw/sample-metadata.tsv \
  --o-visualization ../data/results/table.qzv
```

You can download a copy of `table.qzv` [here](https://github.com/SmithsonianWorkshops/2019-03-05-metabarcoding/raw/master/Metabarcoding_with_Qiime/qiime_artifacts/table.qzv)

Understanding `table.qzv`:
- "Frequency" refers to the number of usable sequences from the sample.
- The "Overview" tab gives a report of the number of reads per sample.
- The "Interactive Sample Detail" uses your metadata to explore the sequencing success.
- The "Feature Detail" gives the unique identifier for each ASV, how many sequences were found of it (pooled amongst sample), and the number of samples it was found it.


```
qiime feature-table tabulate-seqs \
  --i-data ../data/working/rep-seqs-dada2.qza \
  --o-visualization ../data/results/rep-seqs.qzv
```

You can download a copy of `rep-seqs.qzv` [here](https://github.com/SmithsonianWorkshops/2019-03-05-metabarcoding/raw/master/Metabarcoding_with_Qiime/qiime_artifacts/table.qzv)

`rep-seqs.qzv` lists every ASV:
- The ASV's unique identifier.
- The ASV's sequence hyperlinked to an NCBI blastn search.

We now have all the ASVs in `rep-seqs-dada2.qza` and where each ASV was found `table-dada2.qza`. This along with our sampling metadata, `sample-metadata.tsv`, is what we need to move on to taxonomic classification and downstream analyses!

### Taxonomic classification

To assign taxonomy you need a reference database for your locus and the taxonomic groups that you're focusing on. In this study we're using a region of COI for a broad taxonomic assessment of marine species present.

There are different methods to assign taxonomy. The one we'll be using is a supervised machine learning approach called a [Naive Bayes Classifier](https://en.wikipedia.org/wiki/Naive_Bayes_classifier). It uses an input set of sequneces and their taxomoic assignment and then uses the model to assign taxonomy to the un-classified sequences in our samples. There are two-phases to using the classifier, first you train it with `fit-classifier-naive-bayes` and then you use the trained classifier with `classify-sklearn`.

We will be using the [Midori Reference](http://reference-midori.info/) database for COI. This project uses data from GenBank's releases. They produce sets of sequences and taxonomy for different mitochondrial loci. These datasets are pre-formatted to be used as input for different meta-barcoding pipelines. We'll be using the QIIME formatted files.

They produce sets of sequence files for each locus:

- `LONGEST`: only one sequence per species, the representative with the longest sequence is used.
- `UNIQ`: every unique sequence for each species is included.
- `SP`: includes taxa that have genus level identification, but not species-level.

Their datasets based on GenBank release 248 (Feb 2022) have this many sequences included (smallest to largest):

- `MIDORI_LONGEST_NUC_GB248_CO1_QIIME.fasta`: 203165
- `MIDORI_LONGEST_SP_NUC_GB248_CO1_QIIME.fasta`: 780172
- `MIDORI_UNIQ_NUC_GB248_CO1_QIIME.fasta`: 1478887
- `MIDORI_UNIQ_SP_NUC_GB248_CO1_QIIME.fasta`: 2632164

For this workshop we'll be using `MIDORI_LONGEST_NUC_GB248_CO1_QIIME.fasta` as it is the smallest and will require the least time/memory to prepare for taxonomic identification.

#### Training the classifier
#### Download and import Midori CO1 sequences and metadata to QIIME2

```
cd data/raw
wget http://reference-midori.info/download/Latest_GenBankRelease248/QIIME/longest/MIDORI_LONGEST_NUC_GB248_CO1_QIIME.fasta.gz
gunzip MIDORI_LONGEST_NUC_GB248_CO1_QIIME.fasta.gz
wget http://reference-midori.info/download/Latest_GenBankRelease248/QIIME/longest/MIDORI_LONGEST_NUC_GB248_CO1_QIIME.taxon.gz
gunzip MIDORI_LONGEST_NUC_GB248_CO1_QIIME.taxon.gz
cd ../..
```

```
qiime tools import \
  --type 'FeatureData[Sequence]' \
  --input-path ../data/raw/MIDORI_LONGEST_NUC_GB248_CO1_QIIME.fasta \
  --output-path ../data/working/midori_longest_GB248.qza
```

`FeatureData[Sequence]` is how you import fasta data.

```
qiime tools import \
  --type 'FeatureData[Taxonomy]' \
  --input-format HeaderlessTSVTaxonomyFormat \
  --input-path ../data/raw/MIDORI_LONGEST_NUC_GB248_CO1_QIIME.taxon \
  --output-path ../data/working/midori_taxonomy_GB248_taxonomy.qza
```

`HeaderlessTSVTaxonomyFormat` is a tab seperated file that doesn't have a header line. The taxonomy file is formated with the sequence ID and then the taxonomy separated by semicolons.


#### Train the classifier - note that this step takes a lot more RAM than any previous jobs (try a few values and see what works).

```
qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads ../data/working/bold.qza \
  --i-reference-taxonomy ../data/working/ref-taxonomy.qza \
  --o-classifier ../data/working/bold_classifier.qza \
  --verbose
```

### Classifying the sequences

```
qiime feature-classifier classify-sklearn \
  --i-classifier ../data/working/bold_classifier.qza \
  --i-reads ../data/working/rep-seqs-dada2.qza \
  --o-classification ../data/working/bold_taxonomy_results.qza
```

And now we visualize these results

```
qiime metadata tabulate \
  --m-input-file ../data/working/bold_taxonomy_results.qza \
  --o-visualization ../data/working/bold_taxonomy_results.qzv
```

### Generate a tree for phylogenetic diversity analyses
1. alignment with mafft

```
 qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences ../data/working/rep-seqs-dada2.qza \
  --o-alignment ../data/working/aligned-rep-seqs.qza \
  --o-masked-alignment ../data/working/masked-aligned-rep-seqs.qza \
  --o-tree ../data/working/unrooted-tree.qza \
  --o-rooted-tree ../data/working/rooted-tree.qza
```

2. Export your tree to Newick format

```
qiime tools export \
--input-path ../data/working/rooted-tree.qza \
--output-path ../data/results/rooted-tree.tre
```

###  Alpha diversity
1. Use `core-metrics`, which rarefies a FeatureTable to a user-specified depth, and then computes a series of alpha and beta diversity metrics.

```
qiime diversity core-metrics-phylogenetic \
  --i-phylogeny ../data/working/rooted-tree.qza \
  --i-table ../data/working/table-dada2.qza \
  --m-metadata-file ../data/working/sample-metadata.tsv \
  --p-sampling-depth 1109 \
  --output-dir core-metrics-results
 ```

 There are lots of outputs here: look at them with `ls` and with the QIIME visualizer. Also see the QIIME tutorial webpage for further details.
