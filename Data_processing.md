Target-enrichment data proccessing
------------------------
Following steps are meant to be run on the Smithsonian Institution HPC (Hydra). For more information on how to submit jobs to the Hydra cluster see the instructions [here](https://github.com/SmithsonianWorkshops/Hydra-workshop).

### Short version

1. Remove extra text from names using `rename` or `mv` command if your data files are like this: `1760FL-02-14-167_S0_L006_R1_001.fastq.gz` (e.g. `for f in *.fastq.gz; do mv "$f" "${f/1760FL-02/}"; done`)
2. Rename the `fastqc.gz` files based on the species names using `mv` command and name list
3. Trim the files with `trimmomatic` using `trimmomatic.job`
4. Evaluate the reads with `fastqc` using `fastqc.job`
5. Unzip `fastqc.gz` files using `tar` or `gunzip`. For the large files, I recommend using [Pigz](https://www.zlib.net/pigz/) with pthreads option `-p` and sending job(s) rather than unzipping from the login node as it might slow down the login node.
6. Run the `HybPiper/reads_first.py` script. Use `while` command to run multiple files at once.
7. Run the `HybPiper/intronerate.py` to get intron sequences.
8. Run the `HybPiper/retrieve_sequences.py` script to get gene sequences
9. Run `MAFFT` to align the sequences.
10. Run `TrimAl` to trim the alignments.
11. Run `RAxML` to generate gene trees.
12. Concat the gene trees using `cat` command, each tree in a seperate line.
13. Run `ASTRAL` to build the species tree. ASTRAL is a java application, so its better to run it in the local computer rather than sending job to the cluster. It's very fast so you can run in a laptop too!



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
Use Trimmomatic to trim adapters and low quality reads before assembly with following job file:

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
   runtrimmomatic PE -threads $NSLOTS Camptosema_ellipticum_TTGCGAGA_L001_R1.fastq.gz \
   Camptosema_ellipticum_TTGCGAGA_L001_R2.fastq.gz Camptosema_ellipticum_forward_paired-new.fq.gz \
   Camptosema_ellipticum_forward_unpaired-new.fq.gz Camptosema_ellipticum_reverse_paired-new.fq.gz \
   Camptosema_ellipticum_reverse_unpaired-new.fq.gz ILLUMINACLIP:TruSeq3-PE.fa:2:30:10 LEADING:5 TRAILING:15 SLIDINGWINDOW:4:15 MINLEN:36
   #
   echo = `date` job $JOB_NAME done
   ```
* **Notes:** Adapters are listed in the `TruSeq3-PE.fa` file (adjsut accordingly for your platform). Trimmomatic commands like LEADING, TRAILING, SLIDINGWINDOW & MINLEN can be adjusted accordingy.
* **ILLUMINACLIP:TruSeq3-PE.fa:2:30:10** Remove adapters using `TruSeq3-PE.fa` file.
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
* Check HTML output file in a browser like Firefox.

### 3. Running HybPiper pipeline
[HybPiper](https://github.com/mossmatters/HybPiper) is a suite of Python scripts that uses bioinformatics tools in order to extract target sequences from target-enriched reads. All bioinformatics modules need to be loaded via job file, or you can load them from the login node manually. The 'all-genes.fas' is a reference sequences that probes (baits) designed based upon it and HybPiper will map reads to this reference. It requires to be in a specific format. Sine the pipeline use SPAdes assembler, the job file set to run in the high memory nodes (himem). Maximum CPU in this case is 16. It's possible to use Velvet assembler instead of SPAdes. I used SPAdes in this example.

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
To run the script for multiple samples, you can use this command and recall the sample names from the file `namelist.txt`.

`while read name; do ./reads_first.py -b all-genes.fas -r $name*.fastq --prefix $name --bwa --cpu $NSLOTS; done < namelist.txt`


If you want to recover  Chloroplast or Mitochondrial genes, use `./reads_first.py' same as above, but organellar genomes as a reference. I used soybean Chloroplast genome (GenBank: **DQ317523**) as a reference. Use mitochondrial genome of closest lineage to recover mitochondrial genome too.


## Targeted loci recovery heatmap
Using `get_seq_lengths.py` script you can get an idea of how many targeted genes are recovered. Then you can make a heatmap using `gene_recovery_heatmap.R` script in R and `seq_lengths.txt` output file. For more detail see [here](https://github.com/mossmatters/HybPiper/wiki/Tutorial).

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -q sThC.q
#$ -cwd
#$ -j y
#$ -N seq_lengths
#$ -o seq_lengths.log
#
# ----------------Modules------------------------- #
module load bioinformatics/biopython
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
#
python ./get_seq_lengths.py all-genes.fas namelist.txt dna > seq_lengths.txt
#
echo = `date` job $JOB_NAME done
```
Here is a heatmap of recovered genes (x axis) for 25 species.

![example-heatmap](https://user-images.githubusercontent.com/13125143/35326179-c6a76388-00ed-11e8-957a-ac855fc31c1a.jpg)

## Geting supercontig sequences
After mapping the reads to the reference, you can obtain target sequences (targeted genes) using following job file. You need to run job file from the directory where output folder of HybPiper located. In this example, current directory `.`. If you want to get intron sequences as well, you need to run `intronerate.py`script (see below) and then use `supercontig` instead of `dna`.


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
python ./retrieve_sequences.py ./all-genes.fas . dna
#
echo = `date` job $JOB_NAME done
```

To obtain introns run this script in the folder where all your samples are located. 

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -q sThC.q
#$ -l mres=2G
#$ -cwd
#$ -j y
#$ -N introns
#$ -o introns.log
#
# ----------------Modules------------------------- #
module load bioinformatics/biopython
module load bioinformatics/blast
module load bioinformatics/bwa
module load bioinformatics/spades/3.11.1
module load bioinformatics/exonerate
module load bioinformatics/samtools
module load tools/gnuparallel/20160422
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
python ./intronerate.py --prefix Camptosema_ellipticum
#
echo = `date` job $JOB_NAME done
```

### 4. Species tree reconstruction
There are multiple programs to infer species trees from gene trees. For example, [ASTRAL](https://github.com/smirarab/ASTRAL) is one of the statistically consistent summary methods to get species tree from gene trees. Gene trees can be obtained by RAxML or FastTree, then concatinated into a single file by `cat` command, each gene tree on a separate line in newick format. `-i` input file, `-o` name of output file, `2>` writes stdout to the file (recommended). To run ASTRAL, you need to have [Java](https://java.com/).
```
java -jar astral.5.5.2.jar -i genetrees.tre -o genetrees-astral.tre 2> astral.log
```
Use `-q` option to get the scores for the quartets in each node. You can use `-t` option too (e.g. -t 2, -t 4 ...).
```
java -jar astral.5.5.2.jar -q genetrees-astral.tre -i genetrees.tre -o genetrees-astral-scored.tre 2> astral-scored.log
```
Check the .log file, to see how many trees have missing taxa. Also, check "normalized quartet score," which vary between 0-1, the higher score represents less discordant on gene trees. However, this is not a direct assessment of the discordance on each node among gene trees! 

### 5. Get summary of the targeted genes using the [AMAS](https://github.com/marekborowiec/AMAS). 

The following command write alignments summary such as alignments length, variable sites, etc to the `summary.txt` file. `-f` input file format in fasta. AMAS can handle nexus and phylip format too. `-d` dna or `-aa` for amino acid, `*.fas` calculate for all files with `.fas` extension, `-c` number of cores (CPU). You need Python 3 installed. I recommend installing Python 3 using [Miniconda](https://conda.io/miniconda.html). Also use `pip install biopython` to install Biopython, which usually is the latest version.

```
python ./AMAS.py summary -f fasta -d dna -i *.fas -c 6
```

### 6. Assessing Paralogs

HybPiper includes the script `paralog_retriever.py` which collect all paralogs from each sample in `name.txt`, along with all coding sequences from samples without paralogs. If you have a list of genes `gene-list.txt` for which you want to assess paralogs, you can use GNU Parallel with this command:

```
cat gene-list.txt | parallel -k "python ./paralog_retriever.py name.txt {} > {}.paralogs.fasta" 2> paralogs.txt
```
### 7. Clean up
Use `cleanup.py` script from HybPiper to remove thousands of unncessary files, mainly the output of SPAdes assembler. There is a file number limit on Hydra cluster for each user, so this job needs to be done regularly.

```
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 2
#$ -q sThC.q
#$ -cwd
#$ -j y
#$ -N cleanup
#$ -o cleanup.log
#
# ----------------Modules------------------------- #
module load bioinformatics/biopython
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#

python ./cleanup.py Camptosema_ellipticum

#
echo = `date` job $JOB_NAME done
```
