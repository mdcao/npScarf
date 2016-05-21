-------------------------------------------------------------------------
*npScarf*: a module for real-time scaffolding and finishing genomes by using Nanopore sequencing reads
-------------------------------------------------------------------------

*npScarf* (jsa.np.gapcloser) is a program that connect contigs from a draft genomes 
to generate sequences that are closer to finish. These pipelines can run on a single laptop
for microbial datasets. In real-time mode, it can be integrated with simple structural 
analyses such as gene ordering, plasmid forming.

npScaffolder is bundled within the [Japsa package](http://mdcao.github.io/japsa/).

Usage
=====
A summary of *npScarf* usage can be obtained by invoking the --help option::

   	jsa.np.gapcloser --help
Input
------
*npScarf* takes two files as required input::

	jsa.np.gapcloser -s <*draft*> -b <*bam*>
	
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

   	jsa.np.gapcloser -realtime [options]

In this mode, the <*bam*> file will be processed block by block. The size of block 
(number of BAM/SAM records) can be manipulated through option *-read* and *-time*.

The idea of streaming mode is when the input <*nanopore*> file is retrieved in stream.
npReader is the module that provides such data from fast5 files returned from the real-time
base-calling cloud service Metrichor. Ones can run::

	jsa.np.f5reader -realtime -folder c:\Downloads\ -fail -output - | \

	bwa mem -t 10 -k11 -W20 -r10 -A1 -B1 -O1 -E1 -L0 -a -Y -K 3000 <*draft*> - 2> /dev/null | \ 

	jsa.np.gapcloser --realtime -b - -seq <*draft*> > log.out 2>&1

or if you have the whole set of Nanopore long reads already and want to emulate the 
streaming mode::

	jsa.np.timeEmulate -s 100 -i <*nanopore*> -output - | \

	bwa mem -t 10 -k11 -W20 -r10 -A1 -B1 -O1 -E1 -L0 -a -Y -K 3000 <*draft*> - 2> /dev/null | \ 

	jsa.np.gapcloser --realtime -b - -seq <*draft*> > log.out 2>&1

Note that jsa.np.timeEmulate based on the field *timeStamp* located in the read name line to
decide the order of streaming data. So if your input <*nanopore*> already contains the field,
you have to sort it::

	jsa.seq.sort -i <*nanopore*> -o <*nanopore-sorted*> -sortKey=timeStamp

or if your file does not have the *timeStamp* data yet, you can manually make ones. For example::

	cat <*nanopore*> |awk 'BEGIN{time=0.0}NR%4==1{printf "%s timeStamp=%.2f\n", $0, time; time++}NR%4!=1{print}' \
	> <*nanopore-with-time*> 

Real-time annotation
--------------------
The tool includes usecase for streaming annotation. Ones can provides database of antibiotic
resistance genes and/or Origin of Replication in FASTA format for the analysis of gene ordering
and/or plasmid identifying respectively::

	jsa.np.timeEmulate -s 100 -i <*nanopore*> -output - | \

	bwa mem -t 10 -k11 -W20 -r10 -A1 -B1 -O1 -E1 -L0 -a -Y -K 3000 <*draft*> - 2> /dev/null | \ 

	jsa.np.gapcloser --realtime -b - -seq <*draft*> -resistGene <*resistDB.fasta*> -oriRep <*origDB.fasta*> > log.out 2>&1

Or one can input any annotation in GFF 3.0 format:

	jsa.np.gapcloser --realtime -b - -seq <*draft*> -genes <*genesList.GFF*> > log.out 2>&1

Data Analysis
=============
All dataset used and result genomes after running *npScarf* are available following this [link](insert plz)

The QUAST analysis for the comparison with other methods are also made public [here](insert plz)

License
=======
BSD-like license:

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS “AS IS” AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
