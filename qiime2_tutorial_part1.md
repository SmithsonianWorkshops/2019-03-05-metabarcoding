# QIIME 2 Tutorial

In this tutorial, we will run part of the [QIIME 2](https://docs.qiime2.org/2017.9/tutorials/moving-pictures/) Moving Pictures tutorial on Hydra, adjusted for the COI data generated recently at NMNH. 

**To run QIIME2, you will need to both load the module and activate the environment**
```module load bioinformatics/qiime2/2019.1```

```source activate qiime2-2019.1```

*Copy sample data to your space on Hydra*
1. Change directory to your space on Hydra (e.g. ```cd /pool/genomics/USER```)
2. Make a new directory for the data (e.g. ```mkdir qiime2_tutorial```) and ```cd``` into that new directory.
3. Set up a directory structure for your `qiime2_tutorial` "project." Run `mkdir jobs`, then `mkdir logs`, then `mkdir -p data/raw`, then `mkdir -p data/working`, and finally `mkdir -p data/results`.
    Your directory should look like this:
    ```
    .
    |-- data
    |   |-- raw
    |   |-- results
    |   `-- working
    |-- jobs
    `-- logs
    ```
5. We've downsampled the COI data from the recent wetlab workshop and put it here: ```/data/genomics/workshops/qiime2/data``` 
6. Choose two pairs of sequences and use ```cp``` to copy metadata and sequences to your ```raw``` directory. 
7. Check to see that your data are there with ```ls data/raw```. You should have two ```fq.gz``` files, R1 and R2 for two samples and a ```.tsv```
8. Edit the metadata file to include only the rows for the samples you have chosen.

### Import data into QIIME

All data that is used as input to QIIME 2 is in form of QIIME 2 artifacts, which contain information about the type of data and the source of the data. So, the first thing we need to do is import these sequence data files into a QIIME 2 artifact.

**SAMPLE JOB FILE:**
```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -q sThC.q
#$ -pe mthread 2
#$ -l mres=4G,h_data=4G,h_vmem=4G
#$ -cwd
#$ -j y
#$ -N qiime_test
#$ -o ../logs/qiime_test.log
#
# ----------------Modules------------------------- #
module load bioinformatics/qiime2/2019.1
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
source activate qiime2-2019.1
qiime tools import \
 --type ‘SampleData[PairedEndSequencesWithQuality]’ \
 --input-path ../data/raw \
 --input-format CasavaOneEightSingleLanePerSampleDirFmt \
 --output-path ../data/working/paired-end.qza
#
echo = `date` job $JOB_NAME done
```
Feel free to change the name of the output ```.qza``` file in the job file to something more informative. You can also try running this job via an interactive job using ```qrsh```

Now we'll run a summarize command to look at the quality scores. Either run this within a job file with ```qsub``` or on an interactive job with ```qrsh```
```
qiime demux summarize \
  --i-data ../data/working/paired-end.qza \
  --o-visualization ../data/results/paired-end.qzv
  ```
View demux.qzv here: https://view.qiime2.org/ and have a look at the quality of your sequences.

### Trim primers
Now we'll trim the primers off the sequences with cutadapt, which is available through the QIIME2 module. Run this with a job file and ```qsub``` 
```
qiime cutadapt trim-paired \
 --i-demultiplexed-sequences ../data/working/paired-end.qza \
 --p-cores $NSLOTS \
 --p-front-f GGWACWGGWTGAACWGTWTAYCCYCC \
 --p-front-r TANACYTCNGGRTGNCCRAARAAYCA \
 --o-trimmed-sequences ../data/working/paired-end-primers-trimmed.qza
```
Now we'll run a summarize command to look at the quality scores. Either run this within a job file with ```qsub``` or on an interactive job with ```qrsh```
```
qiime demux summarize \
  --i-data ../data/working/paired-end-primers-trimmed.qza \
  --o-visualization ../data/results/paired-end-primers-trimmed.qzv
  ```
View demux.qzv here: https://view.qiime2.org/ and have a look at the quality of your sequences. From these plots, we'll decide how much to trim the sequences for quality.

### DADA2 quality control
Now we'll trim our sequences for quality and do filter out chimeras. This will take the longest of anything we've done so far, so use ```qsub``` and a job file for this command.
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

### View dada2 results summary
Now we’ll run a summarize command to look at how dada2 filtered our reads.
```
qiime metadata tabulate \
  --m-input-file ../data/working/stats-dada2.qza \
  --o-visualization ../data/results/stats-dada2.qzv
```

Then you will need to scp `stats-data2.qzv` to your computer so that you can view the results at https://view.qiime2.org/.

### Copy full dada2 results to data/working
Copy Rebecca's completed dada2 results (for the entire dataset) to your own `data/working` directory. This command will work if you are in `/pool/genomics/USER/qiime2_tutorial`.
```
cp /data/genomics/workshops/qiime2/rep-seqs-dada2.qza /data/genomics/workshops/qiime2/table-dada2.qza /data/genomics/workshops/qiime2/stats-dada2.qza /data/genomics/workshops/qiime2/sample-metadata.tsv data/working
```
Now we’ll run a summarize command to look at how dada2 filtered our reads.
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
  --o-visualization ../data/results/table.qzv \
  --m-sample-metadata-file ../data/working/sample-metadata.tsv
```

```
qiime feature-table tabulate-seqs \
  --i-data ../data/working/rep-seqs-dada2.qza \
  --o-visualization ../data/results/rep-seqs.qzv
```

### Taxonomic classification
#### Training the classifier
#### Import COI reference database sequences and metadata to QIIME2
```
qiime tools import \
  --type 'FeatureData[Sequence]' \
  --input-path /data/genomics/workshops/qiime2/data/classifier/bold_CO1.fasta \
  --output-path ../data/working/bold.qza
```
```
qiime tools import \
  --type 'FeatureData[Taxonomy]' \
  --input-format HeaderlessTSVTaxonomyFormat \
  --input-path /data/genomics/workshops/qiime2/data/classifier/bold_only.txt \
  --output-path ../data/working/ref-taxonomy.qza
```
#### Train the classifier - note that this step takes a lot more RAM than any previous jobs (try a few values and see what works).
```
qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads ../data/working/bold_CO1.qza \
  --i-reference-taxonomy ../data/working/bold_taxonomy.qza \
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
qiime tools export ../data/working/rooted-tree.qza \
  --output-dir ../data/results
```

###  Alpha diversity
1. Use ```core-metrics```, which rarefies a FeatureTable to a user-specified depth, and then computes a series of alpha and beta diversity metrics. 
```
qiime diversity core-metrics \
  --i-phylogeny ../data/working/rooted-tree.qza \
  --i-table ../data/working/table-dada2.qza \
  --p-sampling-depth 1109 \
  --output-dir core-metrics-results
 ```
 There are lots of outputs here: look at them with ```ls``` and with the QIIME visualizer. Also see the QIIME tutorial webpage for further details.
