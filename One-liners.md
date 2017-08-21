# Useful one-liners
* This list will be updated gradually!

Remove files from current folder based on the list in the file 'list.txt'. Replace 'rm' with 'cp' for copy to another folder!
```
while read -r f; do rm "$f"; done <list.txt
```

Submit many jobs at once to Hydra from current folder based on file extension. Job files 'mafft.job' and 'raxml.job' required to be in the same directory.

MAFFT:
```
for file in *.FNA; do qsub -o mafft-$file.log mafft.job $file; done
```

RAxML:
```
for file in *.fas; do qsub -o raxml-$file.log raxml.job $file; done
```
