Data proccessing
------------------------
Following steps are meant to be run on the Smithsonian Institution HPC (Hydra) following the instructions [here](https://github.com/SmithsonianWorkshops/Targeted_Enrichment/blob/master/phyluce.md) with some tweaks myself. Original tutorial is in the Brant Faircloth's website: http://phyluce.readthedocs.org/en/latest/tutorial-one.html

### 1. Count raw reads 
* This script reads fastq files in gzip format and counts 1/4 of lines as a number of raw reads per file. Use gzcat or zcat based on the Linux distro (gzcat works fine in macOS). Summary of the reads will be written to the `tab-delimited` txt file. If you have paired-end data, (R2 files), numbers will be the same with the R1.

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
To trim adapters and low quality reads before assembly, phyluce's illumiprocessor were used which instead of [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic), runs [Trim Galore!](https://www.bioinformatics.babraham.ac.uk/projects/trim_galore/) because of better capiblity on Hydra. However, in my dataset, Trimmomatic seems to work better, so I'll use Trimmomatic independently using following job file:

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
   runtrimmomatic PE -threads 48 Camptosema_ellipticum_TTGCGAGA_L001_R1.fastq.gz Camptosema_ellipticum_TTGCGAGA_L001_R2.fastq.gz Camptosema_ellipticum_forward_paired-new.fq.gz Camptosema_ellipticum_forward_unpaired-new.fq.gz Camptosema_ellipticum_reverse_paired-new.fq.gz Camptosema_ellipticum_reverse_unpaired-new.fq.gz ILLUMINACLIP:TruSeq3-PE.fa:2:30:10 LEADING:5 TRAILING:15 SLIDINGWINDOW:4:15 MINLEN:36
   #
   echo = `date` job $JOB_NAME done
   ```
* Notes: Adapters are listed in the `TruSeq3-PE.fa` file. Trimmomatic commands like LEADING, TRAILING, SLIDINGWINDOW & MINLEN can be adjusted accordingy. Check the job log file for the `TrimmomaticPE: Completed successfully` to be sure no error in the analysis.

To evalute the trimmed reads, I used [FASTQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) application on Hydra using followin job file:
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
