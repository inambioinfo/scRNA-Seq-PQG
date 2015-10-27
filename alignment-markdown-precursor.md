---
layout: default
---

#Setting up Orchestra

To get on Orchestra you want to connect via ssh, and to turn X11 forwarding on. Turning X11 forwarding will let the Orchestra machines open windows on your local machine, which is useful for looking at the data.

Connect to Orchestra using X11 forwarding:

    $ ssh -X your_user_name@orchestra.med.harvard.edu
Orchestra is set up with separate login nodes and compute nodes. You don't want to be doing any work on the login node, as that is set aside for doing non computationally intensive tasks and running code on there will make Orchestra slow for everybody. Here is how to connect to a compute node in interactive mode, meaning you can type commands:

    $ bsub -Is -q interactive bash
Do that and you're ready to roll. You should see that you are now connected to a node named by an instrument like `clarinet` or `bassoon`.

#Introduction to the data

The raw data we will be using for this part of the workshop lives here `/groups/pklab/scw/scw2015/ES.MEF.data/subset`:

    $ ls /groups/pklab/scw/scw2015/ES.MEF.data/subset

***
> [FASTQ](https://en.wikipedia.org/wiki/FASTQ_format) files are the standard format for sequenced reads, and that is the format you will receive from the sequencing center after they sequence your cDNA libraries.

***

The 4 FASTQ files that we will be working with for this module contain 1000 reads each from ~100 samples. The samples are from a [single-cell RNA-seq experiment](http://genome.cshlp.org/content/21/7/1160.long) where researchers were looking at differences between expression in mouse embryonic fibroblasts (MEF) and embryonic stem (ES) cells from mice. 

***
***
**Note on mutiplexing and demultiplexing**

Libraries prepared from hundreds of single cells can be sequenced in the same lane (multiplexed). During the preparation of these libraries each cell is given a distinct "barcode", a short nucleotide sequence that will enable us to separate them out after sequencing. Illumina provides adaptors with a barcodes, and software that can demultiplex the data for you. So, if you use these Illumina barcodes, the sequencing center will return demultiplexed fastq files to you. 

However, Illumina offers only ~96 distinct barcode combinations (as of March 2015). For single cell work where we are interested in simultaneously sequencing more than 96 cells such as with many of the more recent droplet-based microfluidics approaches, we need additional barcodes. To this end, many groups design their own sets of barcodes; since Illumina's software is unable to use these to separate the samples, you will have to perform demultiplexing after receiving the data from the sequencing center. 

This is outside the scope of this workshop, but it is important to note that this will add an additional step prior to the three steps listed below.

***
***

We'll be taking this small subset of reads and performing the following steps:

1. looking at them to make sure they are of good quality 
* aligning them to the mouse genome 
* producing a table of number of reads aligning to each gene for each sample

The counts table generated from step 3 will be the starting point for the more interesting downstream functional analyses. We'll use these subsetted ES and MEF files to demonstrate the workflow; then, we'll look at pre-computed counts results on a full set of samples for the functional analyses.

#Setup

The first thing we will do is copy the small test data over into your home directory.

    $ cd
    $ mkdir workshop
    $ cd workshop
    $ cp -r /groups/pklab/scw/scw2015/ES.MEF.data/subset .

These commands mean:

* change directories to your home directory (`cd` without anything following it will always bring you to your home directory)
* make a directory (`mkdir`) named workshop
* change into the directory (`cd`) named workshop
* copy (`cp`) the folder `/groups/pklab/scw/scw2015/ES.MEF.data/subset` and everything underneath it (using the `-r`) to the current directory (denoted by a period `.`)


#Quality control

With the start of any data analysis it is important to poke around at your data to see where the warts are. We are expecting single-cell datasets to be extra messy; we should be expecting failures in preparing the libraries, failures in adequately capturing the transcriptome of the cell, libraries that sequence the same reads repeatedly and other issues. In particular we expect the libraries to not be very complex; these are just tags of the end of a transcript and so for each transcript there are a limited number of positions where a read can be placed. We'll see how this feature skews some of the more commonly used quality metrics now.

For RNA-Seq data many common issues can be detected right off the bat just by looking at some features of the raw reads. The most commonly used program to look at the raw reads is [FastQC](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/). To run FastQC on the cluster we have to load the necessary module:

	$ module load seq/fastqc/0.11.3

FastQC is pretty fast, especially on small files, so we can run FastQC on one of the full files (instead of just on the subset). First lets copy one of those files over:

    $ mkdir ~/workshop/fastq
    $ cp /groups/pklab/scw/scw2015/ES.MEF.data/fastq/L139_ESC_1.fq ~/workshop/fastq/
    $ cd ~/workshop/fastq
Now we can run FastQC on the file by typing:

    $ fastqc L139_ESC_1.fq
And look at the nice HTML report it generates with Firefox:

    $ firefox L139_ESC_1_fastqc.html
    
Let's analyse some of the plots: 

* The **per base sequence quality** plot shows some major quality problems during sequencing; having degrading quality as you sequence further is normal, but this is a severe drop off. Severe quality drop offs like this are generally due to technical issues with the sequencer, it is possible it ran out or was running low on a reagent. The good news is it doesn't affect all of the reads, the median value still has a PHRED score > 20 (so 1 in 100 probability of an error), and most aligners can take into account the poor quality so this isn't as bad as it looks.
* More worrying is the non-uniform **per base sequence content** plot. It depends on the genome, but for the mouse if you are randomly sampling from the transcriptome then you should expect there to be a pretty even distribution of GC-AT in each sequence with a slight GC/AT bias. We can see that is not the case at all 
* In the next plot, the **per sequence GC content** plot, has a huge GC spike in the middle. Usually you see plots like these when the overall complexity of the reads that are sequenced is low; by that we mean you have tended to sequence the same sample repeatedly, and have very little diversity.
* That notion is reinforced looking at the **sequence duplication levels** plot. If we de-duplicate the reads, meaning remove reads where we have seen the same exact read twice, we'd throw out > 75% of the data. 
* It is also reinforced by the list of kmers that are more enriched than would be expected by chance; a kmer is just every possible k length mer that is seen in a sequence. We'd expect all of these features if we were sequencing the same sequences repeatedly. One thing we would not expect, however, is the big uptick at the end of the **kmer content** plot; the sequences at the end look like some kind of adapter contamination issue. We'd expect these reads to not align unless we trimmed the adapter sequence off.

What are those sequences? We can search for the reads that have one of those enriched sequences with `grep` (*g*lobally search a *r*egular *e*xpression and *p*rint) which print out every line in a file that matches a search string. grep the L139_ESC_1.fq file like this:

    $ grep ACTTGAA L139_ESC_1.fq
    
You should see a lot of sequences that look like this:
`TTGCTAGATATCAGTTGCCTTCTTTTGTTCAGCTAAGGAACTTGAA`

If we BLAST this sequence to the mouse genome, we come up empty, so it is some kind of contaminant sequence, it isn't clear where it comes from. The protocol listed here doesn't have too many clues either. If we could figure out what these sequences are, it would help troubleshoot the preparation protocol and we might be able to align more reads. As it is, these sequences are unlikely to align to the mouse genome, so they mostly represent wasted sequencing.

***
***
**FastQC reports and the implications for data quality (more information)**

* [This blog post](http://bioinfo-core.org/index.php/9th_Discussion-28_October_2010) has very good information on what bad plots look like and what they mean for your data.
* [This page](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/good_sequence_short_fastqc.html) has an example report from a good dataset.
* Please read [this note on evaluating fastqc results](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/Help/2%20Basic%20Operations/2.2%20Evaluating%20Results.html), before being too alarmed by the red "X"s or the orange "!"s in the FastQC report.

***
***

#Alignment

For aligning RNA-seq reads it is necessary to use an aligner that is splice-aware; reads crossing splice junctions have gaps when aligned to the genome and the aligner has to be able to handle that possibility. There are a wide variety of aligners to choose from that handle spliced reads but the two most commonly used are [Tophat](http://ccb.jhu.edu/software/tophat/index.shtml) and [STAR](https://code.google.com/p/rna-star/). They both have similar accuracy, but STAR is much, much faster than Tophat at the cost of using much more RAM; to align to the human genome you need ~40 GB of RAM, so it isn't something you will be able to run on your laptop or another type of RAM-restricted computing environment. 

For this exercise we will use Tophat, so let's load the module first. 
	
	$ module load seq/tophat/2.1.0

To align reads with Tophat you need three things.

1. The genome sequence of the organism you are working with in FASTA format.
* A gene annotation for your genome in Gene Transfer Format (GTF). For example from UCSC.
* The FASTQ file of reads you want to align.

First you must make an [Bowtie2](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml) index of the genome sequence; this allows the Tophat algorithm to quickly find regions of the genome where each read might align. We have done this step already, so don't type in these commands, but if you need to do it on your own, here is how to do it:

    # don't type this in
    $ bowtie2-build your_fasta_file genome_name

We will be using precomputed indices for the mm10 genome today:

    /groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Sequence/Bowtie2Index/
    /groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Annotation/Genes/genes.gtf

Finally, we've precomputed an index of the gene sequences from the ref-transcripts.gtf file here:

     /groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Annotation/Genes/tophat2_trans/
     
We will use this precomputed index of the gene sequences instead of the GTF file because it is much faster; you can use either and they have the same output.

Now we're ready to align the reads to the mm10 genome. We will align two ESC samples and two MEF samples:

```
$ cd ~/workshop/subset
    
$ bsub -J L139_ESC_2 -W 00:20 -n 2 -q short "tophat -p 2 -o L139_ESC_1-tophat --no-coverage-search --transcriptome-index=/groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Annotation/Genes/tophat2_trans/genes /groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Sequence/Bowtie2Index/genome L139_ESC_1.subset.fastq; mv L139_ESC_1-tophat/accepted_hits.bam L139_ESC_1-tophat/L139_ESC_1.bam"
    
$ bsub -J L139_ESC_2 -W 00:20 -n 2 -q short "tophat -p 2 -o L139_ESC_2-tophat --no-coverage-search --transcriptome-index=/groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Annotation/Genes/tophat2_trans/genes /groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Sequence/Bowtie2Index/genome L139_ESC_2.subset.fastq; mv L139_ESC_2-tophat/accepted_hits.bam L139_ESC_2-tophat/L139_ESC_2.bam"
    
$ bsub -J L139_MEF_49 -W 00:20 -n 2 -q short "tophat -p 2 -o L139_MEF-49_tophat --no-coverage-search --transcriptome-index=/groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Annotation/Genes/tophat2_trans/genes /groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Sequence/Bowtie2Index/genome L139_MEF_49.subset.fastq; mv L139_MEF_49-tophat/accepted_hits.bam L139_MEF_49-tophat/L139_MEF_49.bam"
    
$ bsub -J L139_MEF_50 -W 00:20 -n 2 -q short "tophat -p 2 -o L139_MEF-50_tophat --no-coverage-search --transcriptome-index=/groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Annotation/Genes/tophat2_trans/genes /groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Sequence/Bowtie2Index/genome L139_MEF_50.subset.fastq; mv L139_MEF_50-tophat/accepted_hits.bam L139_MEF_50-tophat/L139_MEF_50.bam"
```

Each of these should complete in about seven to ten minutes. Since we ran them all in parallel on the cluster, the whole set should take about seven to ten minutes instead of 30 - 40. Full samples would take hours. 
***
`-J` names the job so you can see what it is when you run bjobs to check the status of the jobs. 

`-W 00:20` tells the scheduler the job should take about 20 minutes. 

`-q short` submits the job to the short queue.

The syntax of the tophat command is 
`tophat -p <NUMBER OF THREADS> --no-coverage-search --transcriptome-index=<PATH TO TRASNCRIPTOME INDEX> <PATH TO GENOME INDEX> <PATH TO INPUT FASTQ>`

At the end we tack on (after the ";") a command to rename (`mv`) the Tophat output filename `accepted_hits.bam` to something more evocative.
***
This method of submitting one job at a time is fine for a small number of samples, but if you wanted to run a full set of hundreds of cells, doing this manually for every sample is a waste of time and prone to errors. You can run all of these automatically by writing a loop:

```
# don't type this in, it is here for future reference
for file in *.fastq; do
    samplename=$(basename $file .fastq)
    bsub -W 00:20 -n 2 -q short "tophat -p 2 -o $samplename-tophat --no-coverage-search --transcriptome-index=/groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Annotation/Genes/tophat2_trans/genes /groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Sequence/Bowtie2Index/genome $file; mv $samplename-tophat/accepted_hits.bam $samplename-tophat/$samplename.bam"
    done
```

This will loop over all of the files with a `.fastq` extension in the current directory and align them with Tophat. We'll skip ahead now to doing some quality control of the alignments and finally counting the reads mapping to each feature.

# Quality checking the alignments

There are several tools to spot check the alignments, it is common to run [RNA-SeQC](https://www.broadinstitute.org/cancer/cga/rna-seqc) on the alignment files to generate some alignment stats and to determine the overall coverage, how many genes were detected and so on. Another option for a suite of quality checking tools is RSeQC. For real experiments it is worth it to look at the output of these tools and see if anything seems amiss.

# Counting reads with featureCounts

The last step is to count the number of reads mapping to the features are are interested in. Quantification can be done at multiple levels; from the level of counting the number of reads supporting a specific splicing event, to the number of reads aligning to an isoform of a gene or the total reads mapping to a known gene. We'll be quantifying the latter, i.e. the total number of reads that can uniquely be assigned to a known gene; basically looking at the location of read alignment on the genome and putting it together with the location of the gene on the genome (this information is contained in the [GTF](http://mblab.wustl.edu/GTF2.html)/annotation file). There are several tools to do this, we will use [featureCounts](http://bioinf.wehi.edu.au/featureCounts/) because it is very fast and accurate.

<pre>
$ module load seq/subread/1.4.6-p3			#featureCounts is part of the subread package
    
$ featureCounts --primary -a /groups/shared_databases/igenome/Mus_musculus/UCSC/mm10/Annotation/Genes/genes.gtf -o combined.featureCounts L139_ESC_1.subset-tophat/L139_ESC_1.subset.bam L139_ESC_2.subset-tophat/L139_ESC_2.subset.bam L139_MEF_49.subset-tophat/L139_MEF_49.subset.bam L139_MEF_50.subset-tophat/L139_MEF_50.subset.bam
</pre>   
    
***
`--primary` tells featureCounts to only count the primary alignment for reads that map multiple times to the genome. This ensures we do not double count reads that map to multiple places in the genome.
***

We need to massage the format of this file so we can use it. We'll take the first column, which is the gene ids and every column after the 6th, which has the counts of each sample.

    $ sed 1d combined.featureCounts | cut -f1,7- | sed s/Geneid/id/ > combined.counts

This command means *s*team *ed*it (`sed`) the file `combined.featureCounts` by 
***
`1d` deleting the first line

`-f1,7-` keeping the first column and everything from the 7th column on 

`s/Geneid/id/` changing the phrase "Geneid" to "id". 
***
This outputs a file in a format with "I" rows of genes and the "J" columns of samples. Each cell is the number of reads that can be uniquely assigned to the gene "i" in the sample "j". This file is of now ready and in the correct format for loading into R.

