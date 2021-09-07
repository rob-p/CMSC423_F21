---
type: assignment
date: 2021-09-07
title: "Programming assignment 1"
#pdf: /static_files/assignments/asg.pdf
#attachment: /static_files/assignments/asg.zip
#solutions: /static_files/assignments/asg_solutions.pdf
due: 2019-09-21 11:59:59
published: true
---


# Overview

This assignemnt has two components:

(a) A basic sequencing simulator, to help understand the nature of what is "required" to reconstruct a genome.

(b) The greedy shortest common superstring algorithm that we covered in class (lecture 2 and 3).

You will implement a simple "perfect" sequence simulator in part (a), that samples reads of a fixed length from along the genome, sampled at _random_ positions.  For simplicity, we will assume that there are no finite sampling effects.  This means you don't have to simulate e.g. starting with multiple clones of the genome, fragmentation, etc. — rather, you can just repeatedly choose random valid positions along the genome, and extract the substring of the appropriate length.  The more interesting / challenging part of component (a) is just computing the statistics for each simulated sampling run.

In part (b) you will implement the greedy shortest common superstring algorithm that we covered in class that repeatedly looks for the string pair with the largest suffix-prefix overlap, merges the strings, and returns them to the working set.  This algorithm is conceptually very simple, but don't be _too_ fooled by its apparent simplicity, you should set aside some time to make sure that you can produce a reasonably efficient implementation of it (e.g. avoid re-computing suffix-prefix overlaps, only computing necessary overlaps for newly merged strings, etc.).


## Overall structure

You will submit your assignment as a tarball named `CMSC423_F21_A1.tar.gz`.  When this tarball is expanded, it should create a folder named `CMSC423_F21_A1`.  The details of how you structure your "source tree" are up to you, but the following **must** hold (to enable proper automated testing of your programs).

 * There should be a script at the top-level of `CMSC423_F21_A1` called `build.sh`.  This should do whatever is necessary to create 2 executables at the top level (one called `randsim` and one called `scsbler`).  If you're comfortable with Makefiles, this can just call `make`, or it could simply run the commands necessary to compile your programs and copy them to the top-level directory.  You can assume this script is run in a `bash` shell.
 
 * There should be a README.md file in the top level directory.  This README file should contain the following information.
     
     - What language have you written your solution in?
     - What did you find to be the hardest part of this assignment?
     - What resources did you consult in working on this assignment (view this as a form of citation; you shouldn't _copy_ code directly from anywhere in your assignment, but if you consulted other sources please list them here).

 * 4 additional required files with the following specific names:
     
     - test_genome.fa - a simple test genome of your creation (this can just be a random string of A,C,G,T or something more sophisticated if you want).  This genome should be *at least* 5000 characters long, but no more than 100000 characters long.
     - test_reads.fa - the result of running your simulator with `read_len` = 100, `target_depth` = 20, and `genome` = `test_genome.fa`
     - test_stats.tsv - the statistics file that results from running your simlator with the above parameters.
     - test_recon.fa - the output that results from running your toy assembler `scsbler` on `test_reads.fa` with `min_olap` = 20.

    When we run your test data, we should get this output.  **NOTE**: Since both the simulator and SCS algorithm contain "random" calls (to generate read positions for the simulator and to break ties in the SCS algorithm), make sure that you always provide explicit seeds to your random number generator so that you get the expected output when run with the test input.


## Part (a), a basic sequencing simulator

In this part of the assignment, you will write a program that reads in a "genome" (in `FASTA`) format and generates random sequencing "reads" from it.  In addition to the input genome, your program will take as input 2 parameters, (1) the length of reads to simulate and (2) the sequencing "depth".  Your program will then randomly sample reads of the specified length until the requested depth is achieved.  In addition to outputting the reads, it will also output some basic statistics about what was sequenced, and how. **Note**: To check that your implementation is performing reasonably, you may want to compare your statistics under different runs to the Lander-Waterman statistics we covered in class.  Are you getting _roughly_ the expected number of islands over many samples given the parameters?

Your program for this part should be called `randsim`.

### Input 

The input consists of 5 arguments, given in this order:

* `read_len` - the length of the simulated reads for you to generate.
* `target_depth` - the target depth of sequencing.  In other words, you should generate the smallest number (m) of reads for which (`m` * `read_len`) / `genome_length` >= `target_depth`.
* `theta` - the $$\theta$$ value we discussed in the Lander-Waterman statistics (the fraction of the read length that a pair of reads must overlap to constitute an island). This input is only necessary for including the number of islands in the output statistics and has no bearing on how reads are simulated.
* `genome` - the path to a FASTA format file containing the genome from which you will simulate reads.
* `output_stem` - the program will write two output files, one called `output_stem.fa` and the other called `output_stem.stats`.

### Output 

#### Simulated Reads

Your program should output the simulated reads in "FASTA" format.  For each read, the header should be of the format `>read_identifier:start_position:length`, where the `read_identifier` is just a unique serial identifier for the read (numbered starting at 0), the `start_position` is the leftmost position on the underlying genome that is covered by the read, and the `length` is just the length of the read (same as the input parameter).  These reads should be written to a file called `output_stem.fa`.

#### Statistics

Your program should output a tab-separated format file with the following information (using the following keys, each associated with the proper value):

* `num_reads` - number of reads generated by the simulation run
* `bases_covered` - total number of distinct nucleotides of the input covered by at least 1 read
* `avg_depth` - the average coverage (this is just `num_reads` * `read_length` / `genome_length`, and should closely match the input parameter of requested depth)
* `var_depth` - the variance of the per-nucleotide depth across the genome; this is just $$s^2 = \frac{\sum_{i=1}^{n} (x_i - \bar{x})^2}{n-1}$$, where $$x_i$$ is the number of reads covering nucleotide $$i$$ in the genome, and $$\bar{x}$$ is just the `avg_depth`.
* `num_islands` - this is the number of distinct contiguous substrings that are covered some set of reads overlapping by at least $$\theta$$ fraction of their length.  In other words, if every nucleotide of the genome is covered by at least one read, such that a read overlaps by $$\theta \ell$$ with the next simulated read, then there is one island.  If there is a single uncovered gap (regardless of it's length), or a region where a subsequent pair of reads overlap by $$< \theta$$, there are 2 islands.  If there are 2 uncovered gaps, there are 3 islands, etc.

This should be written to a file called `output_stem.stats`.

## Part (b), shortest common superstring assembler

In the second part of the assignment, you will implement the greedy algorithm we discussed in class for shortest common superstring as a toy assmbler (let's call it `SCSbler`).  Your program will take as input a file containing a set of "reads" (in `FASTA` format), and will apply the greedy SCS algorithm to these reads.  It will output the approximate shortest common superstring. To "test" your SCS implementation, you can make use of the simulator you built in part (a).  Specifically, you can use the simulator to simulate reads from a known genome, and then attempt to reconstruct them using `SCSbler`.  **I encourage you to play around with these two programs to explore for yourself the effect of different parameters**.  For example, how do read length and depth affect your ability to assemble a single-string from the simulated data?  How does the length of your assembled string compare to the actual underlying genome?  How do these relationships change if the input genome to part (a) has repeats longer than the length of the reads?

Your program for this part should be called `scsbler`.

#### Input

* `reads` - the path to an input file in `FASTA` format containing a set of reads (possibly with duplicates).  In general, you should not assume that the header for each read is in any particular format (i.e. these need not come from your simulator).
* `min_olap` - the minimum suffix-prefix overlap between substrings that will be required to consider merging them in the SCS algorithm.
* `output_name` - the resulting fasta file will be written as a file called `output_name.fa`.

#### Output

* `output_name.fa` - the output file of your program. This file should contain (in `FASTA` format) the strings that result from your implementation of the greedy SCS algorithm.  There should be 1 entry in the `FASTA` file for every distinct string you are able to reconstruct.  The header for each entry should be of the format: `>substring_id:length`, where `substring_id` is a serial identifier (starting from 0) of each distinct substring you are writing out and `length` is the number of nucleotides in this substring.  The overall length of your SCS implementation will be the sum of the lengths of these strings.

