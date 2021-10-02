---
type: assignment
date: 2021-10-01
title: "Programming assignment 2"
#pdf: /static_files/assignments/asg.pdf
#attachment: /static_files/assignments/asg.zip
#solutions: /static_files/assignments/asg_solutions.pdf
due: 2021-10-16 00:01:00
published: true
---


# CMSC 423 Project 2 : Overview

This assignment deals with the construction and querying of the suffix array.  In order to make the construction 
of the suffix array generally useful, it will be necessary to not only build this index, but also to be able to 
save it to disk and load it from disk.  To this end, this project will also require you to be able to serialize 
and deserialize your index.  Your project will consist of 3 executables `buildsa`, `inspectsa`, and `querysa` which 
are described in more detail below.  Though you will implement 3 programs, you can think of the project as being
broken into two parts.

In the first part of the project, you will implement a program to read a reference sequence from a `FASTA` file,
to construct the suffix array on this sequence, and then to write the string and suffix array to file in a binary
format.  You will also implement a program to read the saved file from disk, compute some statistics about 
the suffix array, and write those to a text format output file.

In the second part of the project, you will implement a program to read your serialized suffix array from 
file, as well as to read an input `FASTA` file containing many queries.  Your program will then produce an 
output file with the query results in a well-specified output format.

**Note:** As was mentioned at the start of the class, the projects will generally build off of each other.  Project 1 was a bit of an outlier in this regard.  The only capability from project 1 that you will need is the ability to read `FASTA` files, and the ability to make executables using a `build.sh` script.  However, more overlap will be present between project 2 and future projects, so you should attempt to build as clean and useable of an interface for your code in this project as you can — you will likely be using the suffix array as a library in future projects.

## Overall structure

You will submit your assignment as a tarball named `CMSC423_F21_A2.tar.gz`.  When this tarball is expanded, it should create a
**single** folder named `CMSC423_F21_A2`.  This folder must be created in the directory where the decompression (i.e. `tar xzvf`) is done, and must not be nested inside any other folders. The details of how you structure your "source tree" are up to you, but the following **must** hold (to enable proper automated testing of your programs).

 * There should be a script at the top-level of `CMSC423_F21_A2` called `build.sh`.  This should do whatever is necessary to create 3 executables at the top level (one called `buildsa` and one called `inspectsa` and one called `querysa`).  If you're comfortable with Makefiles, this can just call `make`, or it could simply run the commands necessary to compile your programs and copy them to the top-level directory.  You can assume this script is run in a `bash` shell.
 
 * There should be a README.md file in the top level directory.  This README file should contain the following information.
     
     - What language have you written your solution in?
     - What did you find to be the hardest part of this assignment?
     - What resources did you consult in working on this assignment (view this as a form of citation; you shouldn't _copy_ code directly from anywhere in your assignment, but if you consulted other sources please list them here).

**Turnin** : The assignment turnin will be handled using Gradescope.  We intend to have the infrastructure for this set up by the end of this week, so please check back here for detailed instructions on the submission procedure.

## Part (a), constructing and inspecting the suffix array

In this part of the assignment, you will write two programs.  The first program will be called `buildsa`; it will read in a "genome" (in `FASTA`) format, build the suffix array on this reference, and write the string and suffix array to a binary file.  The second program will be called `inspectsa`; it will read in the binary file written by `buildsa`, and then it will compute some statistics (details below) about the suffix array and write out a test file containing these statistics and the suffix array. 

### `buildsa`

### `buildsa`: Input 

The input consists of 2 arguments, given in this order:

* `reference` - the path to a FASTA format file containing the reference of which you will build the suffix array.
* `output` - the program will write a single binary output file to a file with this name, that contains a serialized version of the input string and the suffix array.

### `buildsa`: Output

Your program will output a file with the name given by the `output` argument above.  This must be a binary file holding 
everything necessary to perform query using your suffix array.  Specifically, it should include the input string itself 
(probably with the sentinel `$` appended) and it should also include an encoding of the entries of the suffix array.
**Note:** The specific binary encoding is up to you — it can be as simple as an integer representing the string 
length followed by the bytes of the string and an integer representing the number of bytes in the suffix array followed
by the entries, or something more complex.  You _are_ allowed to use an external serialization library for this component, 
but the serialization must be to a binary (not text) format.  In C++ you could use something like [cereal](https://uscilab.github.io/cereal/) or [bitsery](https://github.com/fraillt/bitsery); in Rust you could use soemthing like [rkyv](https://github.com/rkyv/rkyv) or [serde](https://github.com/serde-rs/serde) with [bincode](https://github.com/bincode-org/bincode).
For reasons mentioned previously (and for the sake of not running into performance issues in future projects), I'd not recommend using Python for this project, but if you do, you can just use the builtin `pickle` and `cpickle` modules.

**NOTE** : The timeout for all tests for the gradescope server will be 30 minutes.  This will remain the case 
in future projects as the amount of processing you have to do generally increases.  Thus, we _highly_ recommend
that you implement either the O(m lg m) suffix array construction algorithm from the original suffix array 
paper, the O(m) DC3 algorithm we covered in class, or another efficient (i.e. not O($$m^2$$ lg m)) algorithm.

### `inspectsa`
### `inspectsa`: Input 

The input consists of 3 arguments, given in this order:

* `index` - the path to the binary file containing your serialized suffix array (as written by `buildsa` above).
* `sample_rate` - the rate at which suffix array entries will be sampled in your output (see below).
* `output` - the program will write text output to this file containing some basic statistics about the suffix array, as well as a text representation of the suffix array.

### `inspectsa`: Output

Your program will output a file with the name given by the `output` argument above.  The output will consist of text file with exactly 4 lines. Unlike project 1, there are no keys and values; **rather, the values must be written out in precisely the order specified here**.  Your output should contain:

 * `mean LCP1 value` : This is a floating point number representing the _average_ length of the longest common prefix shared between subsequent entries of the suffix array.

 * `median LCP1 value` : This is a floating point number representing the _median_ length of the longest common prefix shared between subsequent entries of the suffix array.

 * `maximum LCP1 value` : This is an integer number representing the _maximum_ length of the longest common prefix shared between subsequent entries of the suffix array.

 * `suffix array spot check` : This is a textual representation of certain values within the suffix array.  Specifically, this line should encode a list of **tab-separated** integers that consists of the following entries of the suffix array (i.e. the entries in the suffix array appearing at the following indices) [`sample_rate`*0, `sample_rate`*1, `sample_rate`*2, ..., `sample_rate`*floor(Suffix Array Length / `sample_rate`)]. 

## Part (b), querying the suffix array

In the second part of the assignment, you will implement 

Your program for this part should be called `querysa`.  This program will take as input 4 arguments, your serialized
suffix array, a `FASTA` file containing queries, a `query mode` parameter, and an `output` file name.  It will perform
query in the SA and report the results in the `output` file specified in the format specified below.

#### `querysa`: Input

* `queries` - the path to an input file in `FASTA` format containing a set of records. **Unlike project 1** you will need to care about both the _name_ and _sequence_ of these fasta records, as you will report the output using the name that appears for a record.  Note, query sequences can span more than one line (headers will always be on one line).
* `query mode` - this argument should be one of two strings; either `naive` or `simpaccel`. If the string is `naive` you
should perform your queries using the `naive` binary search algorithm.  If the string is `simpaccel` you should perform 
your queries using the "simple accelerant" algorithm we covered in class.
* `output` - the name to use for the resulting output.

#### `querysa`: Output

* `output` - the output file of your program. This file should contain the results of your queries in the following format.  Each line should contain a **tab-separated** list containing the following information:

    * `query_name`, `k`, `hit_1`, `hit_2`, `hit_k`

Here, the `query_name` is simply the header of the corresponding FASTA entry (the string after the `>` --- **not including the `>`** on the header line).  The value `k` is the number of occurrences of the query string in the underlying text on which the suffix array is built.  Finally `hit_1` through `hit_k` are the **positions** in the original text (0-indexed) where the query string occurs.  If a query string does not occur in the text, then you should report `k` = 0, and there will be no `hit_1`, ... etc. entries for that query.