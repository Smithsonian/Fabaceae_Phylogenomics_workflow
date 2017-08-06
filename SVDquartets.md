# Running SVDquartets
## What we need:
1. PAUP. You can obtain free test version from [here](http://people.sc.fsu.edu/~dswofford/paup_test/). SVDquartets implemented in the PAUP.
2. Data file in the nexus format with the partition blocks! An example of nexus file with partitions can be obtained from [here](http://www.stat.osu.edu/~lkubatko/data-snakes.nex) from Kubatko et al. (Syst. Biol. 60(4): 393-409, 2011).

You can invoke SVDquartets help by typing 'svdq ?' in the command line of PAUP.

  '''
  paup> svdq ?;

  Usage: SVDQuartets [options...] ;

  Available options:

  Keyword ------- Option type --------------------- Current setting ----------
  evalQuartets    all|random                        random
  nquartets       <real-value>                      20000
  preferAllQ      no|yes                            yes
  speciesTree     no|yes                            no
  partition       <taxpartition-name>               snakespecies
  treeInf         QFM|curTrees|none                 QFM
  seed            <int> or <int:int:int:int>        1234568
  bootstrap       no|standard|multilocus            standard
  loci            <charpartition-name>              (none)
  nreps           <integer-value>                   100
  nthreads        ncpus|<number-of-threads>         3
  mrpFile         <species-outfile-name>            (none)
  qfile           <quartets-outfile-name>           (none)
  qformat         qmc|qfm                           qmc
  replace         no|yes                           *no
  showScores      no|yes                            yes
  showSV          no|yes                            no
  treeFile        <filename-for-bootstrap-treefile> (none)
  treemodel       mscoalescent|shared               mscoalescent
  forceRank       <integer-value>                   0
  keepQuartets    no|yes                            no
  ambigs          missing|distribute                distribute
  plus2           no|yes                            no
  enforce         no|yes                            no
  constraints     <constraint-name>                 (none)

  The following options are for experimental research and are not generally safe/useful::

  Keyword ------- Option type --------------------- Current setting ----------
  qweights        none|reciprocal|exponential       none
  wtlambda        <real-value>                      1000
  lakeInvar       no|yes                            no
  frobDFile       <filename-for-Frobenius-dists>    (none)
                                                   *Option is nonpersistent
'''



There is a great tutorial [here](http://www.stat.osu.edu/~lkubatko/SVDquartets_tutorial2015.html) on running SVDquartets.


