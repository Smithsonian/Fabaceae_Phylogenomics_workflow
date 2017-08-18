# Useful one-liners
* This list will be updated gradually!

Remove files from current folder based on the list in a file 'file.txt'. Replace 'rm' with 'cp' for copy!
```
while read -r f; do rm "$f"; done <file.txt
```

Submit many jobs at once to Hydra from current folder based on file extension

MAFFT:
```
for file in *.FNA; do qsub -o mafft-$file.log mafft.job $file; done
```

RAxML:
```
for file in *.fas; do qsub -o raxml-$file.log raxml.job $file; done
```
