Target-enrichment data proccessing
------------------------
Following steps are meant to be run on the Smithsonian Institution HPC (Hydra). For more information on how to submit jobs to the Hydra cluster see the instructions [here](https://github.com/SmithsonianWorkshops/Hydra-workshop).

### 1. Count raw reads (optional!)
* This script reads fastq files in gzip format and counts 1/4 of lines as a number of raw reads per file. Use gzcat or zcat based on the Linux distro (gzcat works fine in macOS). Summary of the reads will be written to the `tab-delimited` txt file.
   ```
   for f in *R1*_.fastq.gz; do
      zcat "$f" | awk -v fn="$f" -v OFS='\t' 'END{print fn, int(NR/4)}'
   done > raw_reads_summary.txt
   ```
* This is a job file to submit to the Hydra:

   ```
   # /bin/sh
   # ----------------Parameters---------------------- #
   #$ -S /bin/sh
   #$ -q sThC.q
   #$ -l mres=2G,h_data=2G,h_vmem=2G
   #$ -cwd
   #$ -j y
   #$ -N counts
   #$ -o counts.log
   #
   # ----------------Modules------------------------- #
   #
   # ----------------Your Commands------------------- #
   #
   echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
   #
   for f in *R1*_.fastq.gz; do
      zcat "$f" | awk -v fn="$f" -v OFS='\t' 'END{print fn, int(NR/4)}'
   done > raw_reads_summary.txt
   #
   echo = `date` job $JOB_NAME done
   ```
* Summary example:

   ```
   AE108_R1.fastq.gz    9712885
   AE109_R1.fastq.gz    4834828
   AE110_R1.fastq.gz    2988984
   AE111_R1.fastq.gz    5635389
   ```
### 2. Trimming adapters and low quality reads
To trim adapters and low quality reads before assembly, I'll use Trimmomatic using following job file:

   ```
   # /bin/sh
   # ----------------Parameters---------------------- #
   #$ -S /bin/sh
   #$ -pe mthread 48
   #$ -q sThC.q
   #$ -l mres=2G,h_data=2G,h_vmem=2G
   #$ -cwd
   #$ -j y
   #$ -N trim-cael-new
   #$ -o trim-cael-new.log
   #
   # ----------------Modules------------------------- #
   module load bioinformatics/trimmomatic
   #
   # ----------------Your Commands------------------- #
   #
   echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
   echo + NSLOTS = $NSLOTS
   #
   runtrimmomatic PE -threads $NSLOTS Camptosema_ellipticum_TTGCGAGA_L001_R1.fastq.gz Camptosema_ellipticum_TTGCGAGA_L001_R2.fastq.gz Camptosema_ellipticum_forward_paired-new.fq.gz Camptosema_ellipticum_forward_unpaired-new.fq.gz Camptosema_ellipticum_reverse_paired-new.fq.gz Camptosema_ellipticum_reverse_unpaired-new.fq.gz ILLUMINACLIP:TruSeq3-PE.fa:2:30:10 LEADING:5 TRAILING:15 SLIDINGWINDOW:4:15 MINLEN:36
   #
   echo = `date` job $JOB_NAME done
   ```
* Notes: Adapters are listed in the `TruSeq3-PE.fa` file. Trimmomatic commands like LEADING, TRAILING, SLIDINGWINDOW & MINLEN can be adjusted accordingy.
* **ILLUMINACLIP:TruSeq3-PE.fa:2:30:10** Remove adapters using `TruSeq3-PE.f` file.
* **LEADING:5** Remove leading low quality or N bases (below quality 5)
* **TRAILING:15** Remove trailing low quality or N bases (below quality 15)
* **SLIDINGWINDOW:4:15** Scan the read with a 4-base wide sliding window, cutting when the average quality per base drops below 15
* **MINLEN:36** Drop reads below the 36 bases long

Check the job log file for the `TrimmomaticPE: Completed successfully` to be sure no error in the analysis.

To evalute the trimmed reads, use [FASTQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) application following this job file:
   ```
   # /bin/sh
   # ----------------Parameters---------------------- #
   #$ -S /bin/sh
   #$ -q sThC.q
   #$ -l mres=1G
   #$ -cwd
   #$ -j y
   #$ -N fastqc-cael
   #$ -o fastqc-cael.log
   #
   # ----------------Modules------------------------- #
   module load bioinformatics/fastqc
   #
   # ----------------Your Commands------------------- #
   #
   echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
   #
   fastqc ./Camptosema_ellipticum_forward_paired.fq.gz \

   fastqc ./Camptosema_ellipticum_reverse_paired.fq.gz \
   #
   echo = `date` job $JOB_NAME done
   ```
* Check HTML output file in the local browser like Safari, Google Chrome.

### 3. Running HybPiper pipeline
[HybPiper](https://github.com/mossmatters/HybPiper) is a suite of Python scripts that uses bioinformatics tools in order to extract target sequences from target-enriched reads. All bioinformatics modules need to be loaded via job file. The 'all-genes.fas' is a reference sequences that probes (baits) designed based upon it and HybPiper will map reads to this reference. It requires to be in a specific format. Sine the pipeline use SPAdes assembler, the job file set to run in the high memory nodes (himem). Maximum CPU in this case is 16. It's possible to use Velvet assembler instead of SPAdes. I used SPAdes in this example.

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 16
#$ -q mThM.q
#$ -l mres=24G,h_data=24G,h_vmem=24G,himem
#$ -cwd
#$ -j y
#$ -N Camptosema
#$ -o Camptosema.log
#
# ----------------Modules------------------------- #
module load bioinformatics/biopython
module load bioinformatics/blast
module load bioinformatics/bwa
module load bioinformatics/spades/3.10.1
module load bioinformatics/exonerate
module load bioinformatics/samtools
module load tools/gnuparallel/20160422
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
./reads_first.py -b all-genes.fas -r raw-reads/Camptosema_ellipticum_R*.fastq --prefix Camptosema_ellipticum --bwa --cpu $NSLOTS
#
echo = `date` job $JOB_NAME done
```
If you want to recover  Chloroplast or Mitochondrial genes, use `./reads_first.py' same as above, but organellar genomes as a reference. I used soybean Chloroplast genome (GenBank: **DQ317523**) as a reference.

## Geting supercontig sequences
Supercontigs are sequences of exons and introns. After mapping the reads to the reference, you can obtain those target sequences (targeted genes) using following job file. You need to run job file from the directory where output folder of HybPiper located. In this example, current directory `.`.

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -q sThC.q
#$ -cwd
#$ -j y
#$ -N targets
#$ -o targets.log
#
# ----------------Modules------------------------- #
module load bioinformatics/biopython
module load bioinformatics/samtools
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
#
python ./retrieve_sequences.py ./all-genes.fas . supercontig
#
echo = `date` job $JOB_NAME done
```
If you want exon only sequences, use `dna` instead of `supercontig`.

### 4. Species tree reconstruction
There are multiple programs to infer species trees from gene trees. For example, [ASTRAL](https://github.com/smirarab/ASTRAL) is one of the statistically consistent summary methods to get species tree from gene trees. Gene trees can be obtained by RAxML or FastTree. `-i` input file, each gene tree on a separate line in newick format. To run ASTRAL, you need to have Java.
```
java -jar astral.5.5.2.jar -i genetrees.tre -o genetrees-astral.tre 2> astral.log
```
Use `-q` option to get the scores for the quartets in each node. You can use `-t` option too (e.g. -t 2, -t 4 ...).
```
java -jar astral.5.5.2.jar -q genetrees-astral.tre -i genetrees.tre -o genetrees-astral-scored.tre 2> astral-scored.log
```
Check the .log file, to see how many trees have missing taxa. Also, check "normalized quartet score," which vary between 0-1, the higher score represents less discordant on gene trees. However, this is not a direct assessment of the discordance on each node among gene trees! 

### 5. Get summary of the targeted genes using the [AMAS](https://github.com/marekborowiec/AMAS). 

The following command write alignments summary such as alignments length, variable sites, etc to the `summary.txt` file. `-f` input file format in fasta. AMAS can handle nexus and phylip format too. `-d` dna or `-aa` for amino acid, `*.fas` calculate for all files with `.fas` extension, `-c` number of cores (CPU). You need Python 3 installed, `python3` is calling Python 3.

```
python3 ./AMAS.py summary -f fasta -d dna -i *.fas -c 6
```

### 6. Assessing Paralogs

HybPiper includes the script `paralog_retriever.py` which collect all paralogs from each sample in `name.txt`, along with all coding sequences from samples without paralogs. If you have a list of genes `gene-list.txt` for which you want to assess paralogs, you can use GNU Parallel with this command:

```
cat gene-list.txt | parallel -k "python ./paralog_retriever.py name.txt {} > {}.paralogs.fasta" 2> paralogs.txt
```
