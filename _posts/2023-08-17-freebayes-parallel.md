---
title: 'Parallel SNP calling with freebayes'
date: 2023-08-17
permalink: /posts/2023/08/freebayes-parallel/
tags:
  - bioinformatics
  - freebayes
  - SNP
---


Freebayes is a good variant caller and an alternative to the more popular GATK. I have successsfuly used freebayes to call SNPs and INDELs on fungal genomes with good results. However, one of the main drawbacks of freebayes is that it takes a huge amount of time to call variants. This is because freebayes does not take advantage of parallel processing, as a single core is used, and there is no parameter for multithreading.

I recently came across `freebayes-parallel`, which is a `bash` script that comes with freebayes and is designed to allow parallel processing. Here, I describe how to use `freebayes-parallel` within a Snakemake pipeline.

## Using `freebayes-parallel`

Suppose you have all your indexed BAM files and the reference genome in the folder `data/`. BAM file names are in the format `data/<sample>.bam`, and the reference genome is `data/ref.fasta`, like so:


```
data/
	│── ref.fasta
	│── ref.fasta.fai
	│── sample1.bam
	│── sample1.bam.bai
	│── sample2.bam
	│── sample2.bam.bai
	│── samplen.bam
	└── samplen.bam.bai
env_freebayes.yml
Snakefile
```

In the `env_freebayes.yml`, we specify the conda environment, which will contain freebayes, vcflib and tabixpp:

```
channels:
   - conda-forge
   - bioconda
dependencies:
   - freebayes=1.3.6
   - vcflib=1.0.3
   - tabixpp=1.1.0
```

In the `Snakefile ` (provided below), there are two rules:

- Rule `make_bam_fof` will generate a list of BAM files.
- Rule `freebayes` will call `freebayes-parallel` to perform parallel variant calling. Note that the first parameter is a list of regions (chunks) of the genome to be processed simultaneously. This can be obtained with the python script `fasta_generate_regions.py` that comes with freebayes. Here, I am specifying chunks of 1 Mb. The next parameter is the number of CPUs to use. Here, I am specifying 16 CPUs.

```
# get sample names dynamically
SAMPLES, = glob_wildcards("data/{sample}.bam")


rule all:
   input:
      "variants_raw.vcf"


# make a list of BAM files. BAM files must be indexed.
rule make_bam_fof:
   input: 
      expand("data/{sample}.bam.bai", sample=SAMPLES)
   output: 
      "data/bams.fof"    # list of BAM files
   shell: """
      ls data/*.bam > {output}
   """ 
   

# call freebayes-parallel
rule freebayes:
   conda: "env_freebayes.yml"
   input: 
      ref="data/ref.fasta",        # reference genome
      fai="data/ref.fasta.fai",    # index generated with SAMtools faidx
      bams="bams.fof"              # list of BAM files
   output:
      "variants_raw.vcf"
   params:
      chunk=1000000  # size of chunks to be processes in parallel
      cpus=16,       # number of cores to use in parallel
      ploidy=1       # organism is haploid
   shell: """
      freebayes-parallel               \
         <(fasta_generate_regions.py {input.fai} {params.chunk}) {params.cpus} \
         --fasta-reference {input.ref} \
         --ploidy {params.ploidy}      \
         --bam-list {input.bams} > {output}
   """
```

Thus, we can execute the pipeline like so:

```
snakemake -j 1 --use-conda
```




## Some errors I faced before getting it right
While trying to run the pipeline, I came across some errors that were solved by chaning the versions of `vcflib` and `tabixpp`.


In my first try, I used a fresh conda environment:

```
conda create -n freebayes
conda activate freebayes
mamba install -c bioconda freebayes
```

Then tried to run `freebayes-parallel` using my data:

```
freebayes-parallel                      \
   <(${FASTA_FAI} 300000 | head -n+3) 3 \
   --fasta-reference ${FASTA_FAI}       \
   --ploidy 1                           \
   --bam-list ${BAM_LIST} > test.vcf
```

At first I got these error messages, but the process was not killed:

```
vcfuniq: symbol lookup error: /home/azacca/miniconda3/envs/freebayes/bin/../lib/libvcflib.so.1: undefined symbol: wavefront_align
vcfstreamsort: symbol lookup error: /home/azacca/miniconda3/envs/freebayes/bin/../lib/libvcflib.so.1: undefined symbol: wavefront_align
```

However, after a while the process was killed:

```
Traceback (most recent call last):
  File "/home/azacca/miniconda3/envs/freebayes/bin/vcffirstheader", line 10, in <module>
    print(line.strip())
BrokenPipeError: [Errno 32] Broken pipe
```

Searching for a solution online, I found [this](https://github.com/tseemann/snippy/issues/556#issuecomment-1631151656) thread. It appears the error I got was because I was using the most recent version of `vcflib` (`v1.0.9`), and using instead `v1.0.3` could fix it. So, I tried again with a fresh conda environment this time using `vcflib v1.0.3`:


```
conda deactivate
conda env remove -n freebayes
conda create -n freebayes
conda activate freebayes
mamba install -c bioconda freebayes vcflib=1.0.3
```

Once again, I tried to run `freebayes-parallel`:

```
freebayes-parallel                      \
   <(${FASTA_FAI} 300000 | head -n+3) 3 \
   --fasta-reference ${FASTA_FAI}       \
   --ploidy 1                           \
   --bam-list ${BAM_LIST} > test.vcf
```

However, I got similar error messages as before. First I got these error messages:

```
vcfuniq: error while loading shared libraries: libtabixpp.so.0: cannot open shared object file: No such file or directory
vcfstreamsort: error while loading shared libraries: libtabixpp.so.0: cannot open shared object file: No such file or directory
```

Then the process was killed:

```
Traceback (most recent call last):
  File "/home/azacca/miniconda3/envs/freebayes/bin/vcffirstheader", line 10, in <module>
    print(line.strip())
BrokenPipeError: [Errno 32] Broken pipe
```

The error messages above indicate a problem with `tabixpp`. Using the same logic with `vcflib`, I tried to use an older version of the most recent `tabixpp v1.1.2`, specifically `tabixpp v1.1.0`. Thus, I gave another shot:

```
conda deactivate
conda env remove -n freebayes
conda create -n freebayes
conda activate freebayes
mamba install -c bioconda freebayes vcflib=1.0.3 tabixpp=1.1.0
```

And tried to run again `freebayes-parallel`:

```
freebayes-parallel                      \
   <(${FASTA_FAI} 300000 | head -n+3) 3 \
   --fasta-reference ${FASTA_FAI}       \
   --ploidy 1                           \
   --bam-list ${BAM_LIST} > test.vcf
```

This time, no messages, and the output `test.vcf` was produced as expected! Yey!
