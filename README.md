## SANS serif optimization
### 2023 Spring project, Bioinformatics Institute

Students: 
- Zaur Kambiev, Bioinformatics Institute
- Rustam Basyrov, Bioinformatics Institute

Supervisor:
- Kirill Antonets, All-Russia Research Institute for Agricultural Microbiology

### Introduction
[SANS serif](https://gitlab.ub.uni-bielefeld.de/gi/sans) is a tool for whole-genome-based phylogeny estimation based on k-mers in assembled genomes / reads, or coding sequences / amino acid sequences. It doesn’t use multiply alignment to tree reconstruction, but still takes a lot of time to work. In the our research, we aimed to find out bottlenecks in tool execution and tried to optimize them.

![](./plots/sans_workflow.png)

### Analysis
Profiling by [Gperftools CPU Profiler](https://github.com/gperftools/gperftools) (version 2.10) showed the following distribution of running time between the main stages of the algorithm (reading k-mers and weighting splits).

![](./plots/original_distribution.png)

Analyzing the code, we found two problems:
- Reading input files is done in one thread.
- The final sorted list of splits is created simultaneously with filling the hash table of splits. This is time consuming, which is especially noticeable if the length of the resulting list is not limited by the corresponding input argument (-t).

It is on these issues that we have focused our attention.

### Results

1. We moved the formation of the final list of splits into a separate procedure, which goes after filling the hash table of splits. This stabilized the running time of splits’ weighting, and it began to take less than 4% of the total program running time.

![](./plots/distribution_weighting.png)

The modified version of SANS serif is available in the main branch.

2. Three different approaches to file processing parallelization using [OpenMP](https://www.openmp.org) were proposed, implemented and compared on dataset of 200 *Bacillus* genomes.

![](./plots/parallel.png)

In the first approach, the files are processed one by one, and the processing of one file is parallelized.

In the second approach, the files are processed in parallel, and each thread populates its own k-mer hash table. The thread processes the files in batches and after each batch transfers the data to the global hash table.

In the third approach, the threads first process all the files that were distributed to them, and then, in several iterations, the local hash tables of the threads are merged into one big one. The hash tables are merged in pairs, so their number is halved at each iteration.

The second and third approaches, as the most applicable, are available in branches parallel_batch (the second approach) and parallel_mergers (the third approach). Splits' weighting optimization has also been added to these branches.

### Requirements
For the main program, there are no strict dependencies other than C++ version 14.

### Compilation

```
cd <SANS directory>
make
```
By default, the installation creates:
* a binary (*SANS*)

In the *makefile*, two parameters are specified:

* *-DmaxK*: Maximum k-mer length that can be chosen when running SANS.  Default: 32
* *-DmaxN*: Maximum number of input files for SANS. Default: 64

These values can simply be increased if necessary. To keep memory requirements small, do not choose these values unnecessarily large.
    
### Usage

In modified versions, the following input arguments are available:

```
Usage: SANS [PARAMETERS]

  Input arguments:

    -i, --input   	 Input file: file of files format
                  	 Either: list of files, one genome per line (space-separated for multifile genomes)
                  	 Or: kmtricks input format (see https://github.com/tlemane/kmtricks)

  Output arguments:

    -o, --output  	 Output TSV file: list of splits, sorted by weight desc.

    -N, --newick  	 Output Newick file
                  	 (only applicable in combination with -f strict or n-tree)

    (at least --output or --newick must be provided, or both)

  Optional arguments:

    -k, --kmer    	 Length of k-mers (default: 31, or 10 for --amino and --code)

    -t, --top     	 Number of splits in the output list (default: all).
                  	 Use -t <integer>n to limit relative to number of input files, or
                  	 use -t <integer> to limit by absolute value.

    -m, --mean    	 Mean weight function to handle asymmetric splits
                  	 options: arith: arithmetic mean
                  	          geom:  geometric mean
                  	          geom2: geometric mean with pseudo-counts (default)

    -f, --filter  	 Output (-o, -N) is a greedy maximum weight subset (see README)
                  	 options: strict: compatible to a tree
                  	          weakly: weakly compatible network
                  	          n-tree: compatible to a union of n trees
                  	                  (where n is an arbitrary number, e.g. 2-tree)

    -n, --norev   	 Do not consider reverse complement k-mers

    -M, --maxN    	 Compare number of input genomes to compile paramter DmaxN
                  	 Add path/to/makefile (default is makefile in current working directory).

    -v, --verbose 	 Print information messages during execution
  
```

A more detailed description of the arguments is available in [the original repository](https://gitlab.ub.uni-bielefeld.de/gi/sans).

Additional flags are available in versions with parallelization:
```
    -p, --parallel   Number of threads (default: 1)

    -b, --batch      Batch size (only in the batch parallelization, default and maximum: 64)
```

### Examples

1. **Determine splits from assemblies or read files**
   ```
   SANS -i list.txt -o sans.splits -k 31
   ```

2. **Running in 4 threads.**
   ```
   SANS -i list.txt -o sans.splits -k 60 -p 4
   ```
   
2. **Running with batch size 10**
   ```
   SANS -i list.txt -o sans.splits -k 60 -p 4 -b 10
   ```

### References

1. Andreas Rempel , Roland Wittler, SANS serif: alignment-free, whole-genome-based phylogenetic reconstruction, Bioinformatics, Volume 37, Issue 24, December 2021, Pages 4868–4870, https://doi.org/10.1093/bioinformatics/btab444
2. Google Performance Tools, https://gperftools.github.io/gperftools/cpuprofile.html
3. Chandra, R., Dagum, L., Kohr, D., Menon, R., Maydan, D., & McDonald, J. (2001). Parallel programming in OpenMP. Morgan kaufmann.
