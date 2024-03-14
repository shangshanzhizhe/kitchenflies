## Filter for biallelic SNP sites only 

First warning I found, was I couldn't use vcftools because there were polyploids! Then after running rtg-stats, I found that ~0.8% of my SNPs were polyploid. This could be for many reasons; I first thought that there are polyploid individuals in Drosophila (XXY females, some chromosomal abnormalities), despite being a diploid organism, but this percentage seems too high. Other reasons could be sequencing error and/or contamination. On second thought, it actually might be that the poolsnp script had a format that misclassified multiallelic sites as polyploids. When looking at the data again, it seems like we could just 

Long story short, some programs (i.e. SNPRelate) have some "biallelic only" arguments, but it feels like this might be a necessary filtering step for me now. 

So I used `vcftools --gzvcf fileName.vcf.gz --recode-INFO-all --max-alleles 2 --min-alleles 2 -- recode --out output`

or: 

Take the vcf.gz file you made - its not in the right format (needs to be indexed) so:

`bgzip -d file_name.vcf.gz`

The re-zip it: `bgzip -c file_name.vcf > file_name.vcf.gz`

Then index: `bcftools index file_name.vcf.gz`

Then filter for biallelic sites:
bcftools view --max-alleles 2 input.vcf.gz
or 
bcftools view -m2 -M2 -v snps input.vcf.gz > output.vcf.gz (I used this one) 

Then prune: 

bcftools +prune -l 0.25 -w 1000 input.bcf -Ob -o output.bcf
though it seems that the -l has been replaced by -m in the newer version..... ughhh and then do i prune every 1000 or 10000?? 



but then also
bcftools +prune -w 10000bp -n 1 -N 1st
bcftools +prune -w 10000bp -n 1 -N maxAF
bcftools +prune -w 10000bp -n 1 -N rand

where
I added a new option

-N, --nsites-per-win-mode STR 
where STR can be

  maxAF .. keeps sites with biggest AF  (this is the default)
  1st   .. keeps sites that come first
  rand  .. picks sites randomly

  where the default was removing sites with low AF first but could then cause problems in downstream analyses because it assumed that the thin stes are an unbiased sample of the full sites. 

  see https://github.com/samtools/bcftools/issues/1050

  
----

## Convert VCF to SYNC

Kapun says see Kofler et al. 2011, but they do an mpileup file to sync file format. Anyway, now that we've called our SNPs, cleaned, and annotated them (using two different versions of the D mel annotations), now convert them to a sync file. 

```
python scripts/VCF2sync.py \
--vcf SNPs_clean_ann.vcf.gz \
| gzip > SNPs_clean_ann.sync.gz
```


THE SYNC-FILE
1 2R 26 T 0:14:0:0:0:0 0:14:0:0:0:0
2 2R 27 G 0:0:0:14:0:0 0:0:0:14:0:0
3 2R 28 A 14:0:0:0:0:0 14:0:0:0:0:0
4 2R 29 G 0:0:0:14:0:0 0:0:0:14:0:0
I col 1: reference chromosome
I col 2: position
I col 3: reference character
I col 4: allele counts for first population
I col 5: allele counts for second population
I col n: allele counts for n-3 population
⇒ Allele counts are in the form ”A:T:C:G:N:del”
⇒ the sync-file provides a convenient summary of the allele counts of several populations (there
is no upper threshold of the population number).

# Splitting up the VCF files into individual samples (to look for a batch effect with Fst) and/or merging them! 



then you want to make sure you have your list of sample names:

```
for file in SNPs_clean-ann_6.32_bgzf.vcf.gz; do
  for sample in `bcftools query -l $file`; do
    bcftools view -c1 -Oz -s $sample -o ${file/.vcf*/.$sample.vcf.gz} $file
  done
done
```

And it will separate out each sample into its own vcf.gz file! 

To merge all vcf files they have to be in this format, then `bcftools merge file1.vcf.gz file1.vcf.gz -o combined.vcf.gz`

I merged them, then used `plink`