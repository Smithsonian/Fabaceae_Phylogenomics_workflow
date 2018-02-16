# Useful one-liners
* This list will be updated gradually!

Remove files from current folder based on the list in 'list.txt'. Replace `rm` with `cp` for copy to another folder!
```
while read -r f; do rm "$f"; done < list.txt
```

Submit many jobs at once to Hydra from current folder based on file extension. Job files `mafft.job`, `trimal.job` and `raxml.job` required in the current directory. `-o` is outout for the logs.

MAFFT:
```
for file in *.FNA; do qsub -o mafft-$file.log mafft.job $file; done
```

TrimAl:

```
for file in *.mafft; do qsub -o trimal-$file.log trimal.job $file; done
```

RAxML:
```
for file in *.fas; do qsub -o raxml-$file.log raxml.job $file; done
```

Send jobs based on the list in another file:
```
while read i; do qsub -o raxml-$i.log raxml.job $i; done < raxml-list.txt
```
