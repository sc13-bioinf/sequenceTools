# SequenceTools

[![Anaconda package](https://anaconda.org/bioconda/sequencetools/badges/installer/conda.svg)](https://anaconda.org/bioconda/sequencetools)

(bioconda package available thanks to [apeltzer](https://github.com/apeltzer)!)

This repository contains some programs that I use for processing sequencing data. To install, download stack (https://docs.haskellstack.org/en/stable/README/#how-to-install<Paste>) and run "stack install". If you don't have the Haskell compiler installed, it will tell you to run "stack setup". Please do so. The stack installations won't interfere with a system-wide installation. If everything goes as planned, you should end up having the executables defined in this package under ~/.local/bin.

## pileupCaller

The main tool in this repository is the program `pileupCaller` to sample alleles from low coverage sequence data. The first step is to generate a “pileup” file at all positions you wish to genotype. To do that, here is a typical command line, which restricts to mapping and base quality of 30:

    samtools mpileup -R -B -q30 -Q30 -l <list_of_positions.txt> \
        -f <reference_genome.fasta> \
        Sample1.bam Sample2.bam Sample3.bam > pileup.txt

Important Note: You should definitely use the `-B` flag, which disables base alignment quality recalibration. This mechanism is turned on by default and causes huge reference bias with low coverage ancient DNA data. This flag disables the mechanism.

In the above command line, the file "list_of_positions.txt" should either contain positions (0-based) or a bed file (see samtools manual for details). The output is a simple text file with all positions that could be genotyped in the three samples.

Next, you need to run my tool pileupCaller, which you run like this:

    pileupCaller --randomHaploid --sampleNames Sample1,Sample2,Sample3 \
        --samplePopName MyPop -f <Eigenstrat.snp> \
        -e <My_output_prefix> < pileup.txt

Here, options `--sampleNames` gives the names of the samples that is output in the Eigenstrat `*.ind` file, and and `-–samplePopName` is optional to also give the population names in that file (defaults to `Unknown`, you can also change it later in the output). Then, option `-f` gives an Eigenstrat positions file. This is important because the pileup file only contains sites which could be called in at least one of your samples. In order to later merge your dataset with another Eigenstrat file, pileupCaller will check every position in the other Eigenstrat file to make sure every position is output with the correct alleles and missing genotypes if appropriate. Finally, the -o option gives the output format (here Eigenstrat) and the `-e` option gives the prefix for the `*.ind`, `*.pos` and `*.geno` files. Note that this is specific for Eigenstrat output. For FreqSum output, the results will simply be written out to standard out. Note that without specifying `-e`, the output format will be freqSum format, described [here](https://rarecoal-docs.readthedocs.io/en/latest/rarecoal-tools.html#vcf2freqsum), which is useful for debugging your pipeline, since it's just a single file that is output into the terminal and can therefore easily be inspected.

You can also get some help by typing `pileupCaller -h`, which shows a lot more option, for example the sampling method, minimal coverage and other important options.

Note that you can also fuse the two steps above into one unix pipe:

    samtools mpileup -R -B -q30 -Q30 -l <list_of_positions.txt> \
        -f <reference_genome.fasta> \
        Sample1.bam Sample2.bam Sample3.bam | \
    pileupCaller --randomHaploid --sampleNames Sample1,Sample2,Sample3 \
        --samplePopName MyPop -f <Eigenstrat.snp> \
        -e <My_output_prefix>

There is however an issue here: If you have aligned your read data to a version of the reference genome that uses `chr1`, `chr2` and so on as chromosome names, the resulting Eigenstrat file will be valid, but won't merge with other Eigenstrat datasets that use chromosome names `1`, `2` and so on. I would therefore recommend to strip the `chr` from your chromosome names if necessary. You can do that easily using a little UNIX filter using the `sed` tool. In the full pipeline, it looks like this:

    samtools mpileup -R -B -q30 -Q30 -l <list_of_positions.txt> \
        -f <reference_genome.fasta> \
        Sample1.bam Sample2.bam Sample3.bam | sed 's/chr//' | \
    pileupCaller --sampleNames Sample1,Sample2,Sample3 \
        --samplePopName MyPop -f <Eigenstrat.snp> \
        -o EigenStrat -e <My_output_prefix>
