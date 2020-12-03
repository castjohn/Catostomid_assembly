# Catostomid_assembly
###### Author: Carl St. John
###### Contributors: Nathan Backenstose, Trevor Krabenhoft
Notes and code for White Sucker (Catostomus commersonii assembly). This pipeline uses nanopore longreads and illumina reads. Eventually Hi-C reads will be added to achieve 
chromosome-level resolution. Our general workflow is:
1) Generate draft assemblies using Shasta, Miniasm, and Flye assemblers. 
2) Correct assemblies using a consesnus step if assembler does not include this. Illumina reads are necessary for this step.
3) polish the genome using pilon if a polishing step is not included in assembly program. 
4) more to come

#### Prepare Nanopore reads, fastq -> fasta
```
gunzip -c inputFileName.fastq.gz | awk '{if(NR%4==1) {printf(">%s\n",substr($0,2));} else if(NR%4==2) print;}' > outputFileName #fastq.gz2fasta
```

#### Shasta assembly
Shasta will output an assembly file and a .gfa file. The GFA can be viewed using bandage on your own machine. Try playing with the depth to get a clearer picture of the assembly
```
nohup /home/krablab/Documents/apps/shasta-Linux-0.4.0 --input reads.fasta --command assemble --threads 72 --assemblyDirectory /path/to/directory --Reads.minReadLength 1000 & 
```

#### Miniasm assembly
A minimap run will precede the miniasm assembly. You are essentially mapping the nanopore reads to themselves for indexing purposes. The output will be a massive file so pipeing
directly to zipped file saves space.
```
minimap2 -x ava-ont -t 78 NanoporeReads.fastq.gz NanoporeReads.fastq.gz | gzip -9 all_reads.paf.gz
```
Next we run miniasm to generate an assembly. The output is only a GFA file so to create a fasta assembly file we will take rows 2 and 3 of the gfa
```
miniasm -f NanoporeReads.fastq.gz all_reads.paf.gz > assembly.gfa
awk '$1 ~/S/ {print ">"$2"\n"$3}' assembly.gfa > assembly.fasta
```
note that we use the '>' to output the assembly so if using nohup, do not use the '>' to create a nohup.log file. It's best just to rename the nohup.out file that is auto generated.

#### Flye Assembly

Flye includes a polishing step and you can start at any step in the assembly. 2 polishing steps are recommended as a start the flag for this option is -iterations
```
flye --nano-raw #reads.fasta/fastq --genome-size #genome_size_estimate_(2.5g for white sucker) --threads 78 --iterations #Polishing_iterations_desired_(rec_>2) -o /path/to/output/directory
```
Flye outputs many files including a log so no need to make a nohup log file. Sandve lab follows Flye assembly with polishing with PEPPER then polishing with PILON. Waiting for PEPPER to be installed on server. 

#### Racon

No matter what assembly you use, each benefits or requires 2-3 Racon runs to achieve the best consensus assembly. Use the following command to run racon:
```
minimap2 -t 75 current_assembly.fasta IlluminaReads.fastq/fasta | gzip > assembly_racon1.paf.gz
/home/krablab/Documents/apps/racon/build/bin/racon -t 75 NanoporeReads.fasta/fastq assembly_racon1.paf.gz current_assembly.fasta > assembly_racon1.fasta
```
Recycle this command as you generate runs 2 and 3. Upload each to [gvolante](https://gvolante.riken.jp/) to assess BUSCO gene score. You should choose Busco V2/V3 and the Actinopterygii core genes dataset.

#### Pilon polishing
At this point in the pipeline, you may be able to choose the best assembly oout of your different assembly programs. Select the the assembly_racon#.fasta with the highest BUSCO
score and N50 (There may be a trade-off) to polish with pilon. Use the script 
```
/mnt/krab3/CC5/scripts$ AISO_pilon_shared-csj.sh 
REF=$1 #path to reference assembly to be polished in same folder as analysis
FORWARD=$2 #Path to forward Illumina reads
REVERSE=$3 #Path to reverse Illumina reads
THREADS=$4 #threads to be used in bwa, samtools view, and pilon

```
We run 2 or three iterations of pilon to achieve the best assembly. replace $1 with the most recent assembly. Upload each iteration to gvolante again to assess assembly completeness.


