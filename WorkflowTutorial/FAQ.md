# Frequently Asked Questions

## I used bedtools/bedops to compare genomic coordinates and got less overlap then expected

- It is common to use something like awk to convert genomic coordinates to [bed format](https://genome.ucsc.edu/FAQ/FAQformat.html), then use [bedtools](http://bedtools.readthedocs.io/en/latest/) or [bedops](http://bedops.readthedocs.io/en/latest/content/reference/statistics/bedmap.html) to overlap coordinates with other datasets. You often see a commands like the following used to do this conversion:

```awk '{print $1, $2, $2+1, $3, $4}```

- However, this is usually incorrect. Genomic coordinates are usually reported using one-based numbering (ie; the first base of the chromosome is numbered 1), but bed files use zero-based, half-open coordinates (the first base is represented as 0,1). See [here](http://genome.ucsc.edu/blog/the-ucsc-genome-browser-coordinate-counting-systems) for details. This means that the correct command to convert one-based coordinates to bed format is

```awk '{print $1, $2-1, $2, $3, $4}```

- Using the first command above creates an off-by-one error that may go unnoticed when comparing broad intervals like transcripts or ChIP-Seq peaks, but quickly becomes obvious when working with single-base intervals such as SNPs or coding sequence starts/ends.

## When running a differential gene expression analysis I got an error saying "model matrix is not full rank"/"system is computationally singular"

- This means that the software was unable to convert your sample design to a model matrix because one or more of the factors in your model is a linear combination of other factors.
- This can happen for good reasons (a random effect is nested within the fixed effects, such as individuals nested within genotypes), bad reasons (the model has been mis-specified) or ugly reasons (batch effects confounded with treatment effects).
- In the first instance, the model can be modified by adding an additional dummy variable that cuts across fixed effects and nesting the random effect within this dummy variable. The second instance is simply a coding error. The third instance means a trip back to the bench to rerun the experiment.
- See [here](https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html#model-matrix-not-full-rank) or section 3.5 [here](http://bioconductor.org/packages/release/bioc/vignettes/edgeR/inst/doc/edgeRUsersGuide.pdf) (pdf) for more information.

