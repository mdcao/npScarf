# *npScarf*: Scaffolding and Completing Assemblies in Real-time Fashion

*npScarf* (jsa.np.npscarf) is a program that scaffolds and completes draft genomes assemblies 
in real-time with Oxford Nanopore sequencing. The pipeline can run on a computing cluster
as well as on a laptop computer for microbial datasets. It also facilitates the real-time 
analysis of positional information such as gene ordering and the detection of genes from
mobile elements (plasmids and genomic islands).

## Installation

Dependency: The pipeline requires the following software installed

* SPAdes >= 3.5
* bwa >= 7.11

Quick installation guide::

    $ git clone https://github.com/mdcao/japsa
    $ cd japsa
    $ make install \
       [INSTALL_DIR=~/.usr/local \] 
       [MXMEM=7000m \] 
       [SERVER=true \] 
       [JLP=/usr/lib/jni:/usr/lib/R/site-library/rJava/jri]

*npScarf* module is bundled within the [Japsa package](http://mdcao.github.io/japsa/).
Details of installation (including for Windows) and usage of Japsa can be found 
in its documentation hosted on [ReadTheDocs](http://japsa.readthedocs.org/en/latest/index.html) 
In order to run the *npScarf* in real-time, [npReader]( https://github.com/mdcao/npReader)
and particularly HDF library need to be istalled properly. Please refer to the installation 
instructions for [npReader]( https://github.com/mdcao/npReader) repository.


## Tutorial

This tutorial will walk through how to use *npScarf* to complete a genome assembly
of the K. pnuemoniea ATCC BAA-2146 (Kpn2146) bacterial strain using Illumina
and nanopore sequencing data.

#### Primary data sources: 

1. Illumina sequencing data: It is essential that the reads are trimmed to remove 
all adaptors. Low quality bases should also be removed. We make available the sequencing
data for the Kpn2146 sample, sequenced with Illumina MiSeq and are trimmed
with trimmomatic: [file1](http://data.genomicsresearch.org/Projects/npScarf/data/Kp2146_paired_1.fastq.gz)
and [file 2](http://data.genomicsresearch.org/Projects/npScarf/data/Kp2146_paired_2.fastq.gz).

2. Nanopore sequencing data: The raw data (before base-calling) of the Kpn2146 
can obtained from ENA with run accession ERR868296.


Intermediate data are also made available as you walk through the tutorial.

#### Processing

* Assemble the Illumina data with SPAdes using 16 threads in parallel. Option --careful would help to reduce the errors but the improvement is not so significant. It's safe to exclude it from the command if you want to save the running time.

```
$ spades.py --careful --pe1-1 Kp2146_paired_1.fastq.gz --pe1-2 Kp2146_paired_2.fastq.gz -o spades -t 16
```

The result contigs file of interest is spades/contigs.fasta. The contig list is then sorted with 

```
$ jsa.seq.sort -r -n --input spades/contigs.fasta --output Kp2146_spades.fasta 
```

The assembly of the Illumina data (using SPAdes 3.5) of the Kpn2146 is made available 
[here](http://data.genomicsresearch.org/Projects/npScarf/data/Kp2146_spades.fasta)

* Create the bwa index for the Illumina assembly:

```
$ bwa index Kp2146_spades.fasta
```

* In batch mode where all nanopore data have been sequenced and base-called, the scaffolding can be
done in batch mode with the command:

```  
$ bwa mem -t 10 -k11 -W20 -r10 -A1 -B1 -O1 -E1 -L0 -a -Y Kp2146_spades.fasta Kp2146_ONT.fastq  | jsa.np.npscarf -b - -seq Kp2146_spades.fasta -prefix Kp2146-batch 
```

The nanopore sequencing data for the Kpn2164 sample in fastq format is made available
[here](http://data.genomicsresearch.org/Projects/npScarf/data/Kp2146_ONT.fastq.gz).

* In real-time mode, assuming the base-called data from Metrichor service are stored
in folder Downloads, the pipeline can run with following command:

```
$ jsa.np.npreader  --realtime --folder Downloads --fail --stat --number --output - \
 | bwa mem -t 10 -k11 -W20 -r10 -A1 -B1 -O1 -E1 -L0 -a -Y -K 3000 Kp2146_spades.fasta -  \
 | jsa.np.npscarf -realtime -b - -seq Kp2146_spades.fasta -prefix Kp2146-realtime > log.out 2>&1
```

The processing can be distributed over a network cluster by using the streaming utilities
provided in japsa package. Information can be found  
[here](http://japsa.readthedocs.io/en/latest/tools/jsa.util.streamServer.html) and
[here](http://japsa.readthedocs.io/en/latest/tools/jsa.util.streamClient.html) and 
[examples are here](http://japsa.readthedocs.io/en/latest/tools/jsa.np.npreader.html)



## Detailed Usage

A summary of *npScarf* usage can be obtained by invoking the --help option::

   	jsa.np.npscarf --help
   	
Note: options with dash or dash-dash (GNU style) are all acceptable and equivalent iff no ambiguity is introduced.
For example ones can call instead

	jsa.np.npscarf -help 
	
or even
	
	jsa.np.npscarf -h
	
since h is the only prefix in this command's list of options.

Input
------
*npScarf* takes two files as required input::

	jsa.np.npscarf -s <*draft*> -b <*bam*>
	
<*draft*> input is the FASTA file containing the pre-assemblies. Normally this 
is the output from running SPAdes on Illumina MiSeq paired end reads.

<*bam*> contains SAM/BAM formated alignments between <*draft*> file and <*nanopore*> 
FASTA/FASTQ file of long read data. We use BWA-MEM as the recommended aligner 
with the fixed parameter set as follow::

	bwa mem -k11 -W20 -r10 -A1 -B1 -O1 -E1 -L0 -a -Y <*draft*> <*nanopore*> > <*bam*>
	
Output
------
*npScarf* output is specified by *-prefix* option. The default prefix is \'out\'.
Normally the tool generate two files: *prefix*.fin.fasta and *prefix*.fin.japsa which 
indicate the result scaffolders in FASTA and JAPSA format.

In realtime mode, if any annotation analysis is enabled, a file named 
*prefix*.anno.japsa is generated instead. This file contains features detected after
scaffolding.

Real-time scaffolding
----------------------
To run *npScarf* in streaming mode::

   	jsa.np.npscarf -realtime [options]

In this mode, the <*bam*> file will be processed block by block. The size of block 
(number of BAM/SAM records) can be manipulated through option *-read* and *-time*.

The idea of streaming mode is when the input <*nanopore*> file is retrieved in stream.
npReader is the module that provides such data from fast5 files returned from the real-time
base-calling cloud service Metrichor. Ones can run::

	jsa.np.npreader -realtime -folder c:\Downloads\ -fail -output - | \

	bwa mem -t 10 -k11 -W20 -r10 -A1 -B1 -O1 -E1 -L0 -a -Y -K 3000 <*draft*> - 2> /dev/null | \ 

	jsa.np.npscarf -realtime -b - -seq <*draft*> > log.out 2>&1

or if you have the whole set of Nanopore long reads already and want to emulate the 
streaming mode::

	jsa.np.timeEmulate -s 100 -i <*nanopore*> -output - | \

	bwa mem -t 10 -k11 -W20 -r10 -A1 -B1 -O1 -E1 -L0 -a -Y -K 3000 <*draft*> - 2> /dev/null | \ 

	jsa.np.npscarf -realtime -b - -seq <*draft*> > log.out 2>&1

Note that jsa.np.timeEmulate based on the field *timestamp* located in the read name line to
decide the order of streaming data. So if your input <*nanopore*> already contains the field,
you have to sort it::

	jsa.seq.sort -i <*nanopore*> -o <*nanopore-sorted*> -sortKey=timestamp

or if your file does not have the *timestamp* data yet, you can manually make ones. For example::

	cat <*nanopore*> |awk 'BEGIN{time=0.0}NR%4==1{printf "%s timestamp=%.2f\n", $0, time; time++}NR%4!=1{print}' \
	> <*nanopore-with-time*> 

Real-time annotation
--------------------
The tool includes usecase for streaming annotation. Ones can provides database of antibiotic
resistance genes and/or Origin of Replication in FASTA format for the analysis of gene ordering
and/or plasmid identifying respectively::

	jsa.np.timeEmulate -s 100 -i <*nanopore*> -output - | \

	bwa mem -t 10 -k11 -W20 -r10 -A1 -B1 -O1 -E1 -L0 -a -Y -K 3000 <*draft*> - 2> /dev/null | \ 

	jsa.np.npscarf -realtime -b - -seq <*draft*> -resistGene <*resistDB.fasta*> -oriRep <*origDB.fasta*> > log.out 2>&1

Or one can input any annotation in GFF 3.0 format:

	jsa.np.npscarf -realtime -b - -seq <*draft*> -genes <*genesList.GFF*> > log.out 2>&1
	
Assembly graph
--------------
*npScarf* can read the assembly graph info from SPAdes to make the results more precise (in SNP level).
This function is still on development and the results might be slightly deviate from the stable version in
term of number of final contigs.

## Citation

Please cite npScarf if you find it useful for your research

Cao, M.D., Nguyen, H.S., et al. Scaffolding and Completing Genome Assemblies in Real-time with Nanopore Sequencing. 
Nature Communications 8, Article number: 14515 (2017). doi:[10.1038/ncomms14515].

Data and results from npScarf presented in the paper are made available following 
[this link](http://data.genomicsresearch.org/Projects/npScarf/data).
The QUAST analysis of results from npScarf and competitive methods are in also 
presented for 
[K. pneumoniae ATCC BAA-2146](http://data.genomicsresearch.org/Projects/npScarf/results/QUAST/Kp2146/report.html),
[K. pneumoniae ATCC 13883](http://data.genomicsresearch.org/Projects/npScarf/results/QUAST/Kp13883/report.html),
[E. coli K12 MG1655] (http://data.genomicsresearch.org/Projects/npScarf/results/QUAST/EcK12S/report.html),
[S. Typhil H58] (http://data.genomicsresearch.org/Projects/npScarf/results/QUAST/StH58/report.html)
and 
[S. cerevisae W303] (http://data.genomicsresearch.org/Projects/npScarf/results/QUAST/W303/report.html).


## License

See [Japsa license](https://github.com/mdcao/japsa/blob/master/LICENSE.md)
