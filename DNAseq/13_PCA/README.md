# PCA

population genetics? is it a good idea? questionable. shangzhe says take it with a grain of salt. see https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7750941/ and https://doi.org/10.1101/2021.04.11.439381

The goal of a PCA is to try to cluster samples in how ever many degrees of a "principle component" that explains X% of variation among the samples while reducing noise. It's basically a visual check (and should be used as one) to measure how each variable is assoicated with another in a covariance matrix. It transforms some correlated features in data into orthogonal components. PCA uses eigen-decomposition to breakdown a matrix where we get a diagonal matrix of eigenvalues (the dimension of the matrix, coefficients applied to eigenvectors to give length/magnitude) and a matrix formed by the eigenvectors (unit vectors that assess the directions of the spread of the data). 

# LD Pruning
It seems like there is not really a solid consensus on how pruning should be done. So here are a few options:

1. SNPRelate before using this tool for a PCA


2. Plink pruning

Method A[^1]:

First, do a linkage analysis 

```
plink --gzvcf fileName.vcf.gz 
--double-id \
--allow-extra-chr \
--set-missing-var-ids @:# \
--indep-pairwise 10 10 0.2 \
--out XXX
```

| Command      | Description |
| ----------- | ----------- |
| `gzvcf` | input vcf.gz file |
| `--double-id` | for plink to duplicate sample ID (since plink usually expects both a family and individual ID) |
| `--allow-extra-chr` | allows additional chromosomes beyond the set humans have (plink works with human data usually) |
| `--set-missing-var-ids @:#` | sets a variant ID for SNPs, where the `@` ells u where the chrosome code should go and the `#` where the base-pair position belongs |
| `--indep-pairwise` | the linkage pruning part, with 10 Kb as a window size, 10 as a window step size (move 10 bp each  time you calculate linkage) and 0.2 as an r^2 threshold (of what we want to tolerate) |
| `--out` | the output prefix all your files will have |

This will write out two files we will specifically need, `XXX.prune.in`, `XXX.prune.out`, which shoe us a list of sites which fell below the linkage threshold (the ones we want to keep) and the those that are above the threshold (the ones we want to throw away). 

Next, we want to use the output to produce files that are necessary for a PCA.

```
plink --gzvcf fileName.vcf.gz \
--double-id \
--allow-extra-chr \
--set-missing-var-ids @:# \
--extract XXX.prune.in \
--make-bed
--pca
--out XXX
```

| Newer Commands      | Description |
| ----------- | ----------- |
| `--extract` | input file of positions we wanted to retain before |
| `--make-bed` | writes out additional files for another type of population structure analysis |
| `--pca` | writes out eigenvector and eigenvalue files |

Where you now end up with the eigenvector and eigenvalue files, as well as a `.bed` file is which a binary file for admixture analyes (gives us the genotypes of pruned dataset as 1s and 0s, the `.bim` file which is a map/information file of the variants and a `.fam` file which is a map file for all the individuals in the `.bed` file.

Finally, you might want to keep the plink output as a VCF file, but see the memo below: `plink --bfile [filename prefix] --allow-extra-chr --recode vcf --out [VCF prefix]`



You can also use plink like so:
`plink --vcf SNPs_clean_ann.vcf.gz --maf 0.05 --recode --alow-extra-chr --r2 --ld-window-kb 1 --ld-window 1000 --ld-window-r2 0 --out SNPs_ld`

but I am unsure with what to do with this output (SNPs_ld)

3. Brute force way, using the argument `--thin` and setting it to 10 kb (kept 12,944/2,280,074 sites). This will output a vcf file: `vcftools --ggzvcf fileName.vcf.gz --thin 10000 --recode --out outputPrefix`
   
4. bcftools
- bcftools +prune -m 0.2 -w 10000 input.vcf.gz -Oz -o output.vcf.gz
though it seems that the -l has been replaced by -m in the newer version..... ughhh and then do i prune every 1000 or 10000?? 
- bcftools +prune -w 10000 -n 1 -N 1st
- bcftools +prune -w 10000 -n 1 -N maxAF
- bcftools +prune -w 10000 -n 1 -N rand

with the options for --nsites-per-win-mode STR 
where -N, 
where STR can be  maxAF (keeps sites with biggest AF, default), 1st (keeps sites that come first) and rand (picks sites randomly) 

  where the default was removing sites with low AF first but could then cause problems in downstream analyses because it assumed that the thin stes are an unbiased sample of the full sites. 

 see https://github.com/samtools/bcftools/issues/1050

:memo: Converting plink files does not seem to be a good idea, it kind of messes things up/lose metadata, it seems you may/may not need allele reference data input too. For example, it seemed I lost the sex data, so I tried to add the argument `--update-sex txtFile` and I created a random text file with Year, sample id, sex with tabs in between in notepad... did not work. I think the formatting is off but I'm not sure why...

[^1]:https://speciationgenomics.github.io/pca/
