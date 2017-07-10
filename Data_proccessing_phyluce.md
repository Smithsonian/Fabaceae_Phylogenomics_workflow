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
To trim adapters and low quality reads before assembly, phyluce's illumiprocessor were used but instead of [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic), [Trim Galore!](https://www.bioinformatics.babraham.ac.uk/projects/trim_galore/) were used because of capiblity to run on Hydra cluster.

