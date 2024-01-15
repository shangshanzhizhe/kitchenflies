# Alignment 

Next, we're going to map our trimmed reads to the reference genome. This is the point in which we determine the position in the genome based on the read sequence. For this, we will use BWA[^1]. `bwa mem` is preferable for longer Illumina reads  > 70 bp and < 1Mbp. It also directly produces SAM files. 

The BWA-MEM algorithm performs local alignment. It may produce multiple primary alignments for different part of a query sequence. This is a crucial feature for long sequences. However, some tools such as Picard’s markDuplicates does not work with split alignments. One may consider to use option -M to flag shorter split hits as secondary.

The output of the ‘aln’ command is binary and designed for BWA use only. BWA outputs the final alignment in the SAM (Sequence Alignment/Map) format.

Then we will pipe them into Samtools[^2], filtering with MQ (mapped quality score) > 20. Using `samtools view` views and converts SAM/BAM/CRAM files. 

:exclamation: *Computationally and time consuming* :exclamation:

```
conda install bioconda::bwa

bwa mem \
-M \
./PATH/TO/2_bam_libraries/reference_genome.fa  \
./PATH/TO/1_cutadapt_trimmed/trimmed_read_1.fq.gz  \
./PATH/TO/1_cutadapt_trimmed/trimmed_read_2.fq.gz \

| samtools view \
-Sbh -q 20 -F 0x100 - > ./PATH/TO/2_bam_libraries/samplename_library.bam \
```

| Command      | Description |
| ----------- | ----------- |
| `-M` | Mark shorter split hits as secondary (for Picard compatibility) |
| - | reference genome |
| - | input file 1.fq.gz |
| - | input file 2.fq.gz |
| `-S` | Ignored for compatibility with previous samtools versions. Previously this option was required if input was in SAM format, but now the correct format is automatically detected by examining the first few characters of input. |
| `-b` |Output in the BAM format.|
| `-h` | Includes header in the output|
| `-q` | filtering alignments with MAPQ smaller than 20 |
| `-F` | Do not output alignments with bits set in FLAG present in the FLAG field |
| `0x100` |	SECONDARY	secondary alignment, specified by hex|

Plotting?

gnuplot package 

# Sorting
Using Picard [^3], the SAM file is going to be sorted by coodinate (SortOrder is found in the SAM file header), here read alignments are sorted into subgroups by the reference sequence name (RNAME) field using the reference sequence dictionary (@SQ tag), then secondarily sorted using the left-most mapping position of the read (POS). Doing this by coordinate makes SAM files smaller, visualizing more efficient, and helps when marking duplicates (for paired reads, this is done by looking at 5' mapping positions of both reads) because this guarantees that the reads physically closer inside the SAM file are also close in the genome.
 
```
conda install bioconda::picard

picard SortSam \
I=./PATH/TO/2_bam_libraries/samplename_library.bam \
O=./PATH/TO/3_sortbam_libraries/samplename_library-sort.bam \
SO=coordinate 
```

| Command      | Description |
| ----------- | ----------- |
| `-I` | input library .bam |
| `-O` | output sorted library .bam |
| `SO` | Sort order of output file. |



[^1]: Li H. and Durbin R. (2009) Fast and accurate short read alignment with Burrows-Wheeler Transform. Bioinformatics, 25:1754-60. [PMID: 19451168] <https://github.com/lh3/bwa>
[^2]: Danecek, P., Bonfield, J. K., Liddle, J., Marshall, J., Ohan, V., Pollard, M. O., ... & Li, H. (2021). Twelve years of SAMtools and BCFtools. Gigascience, 10(2), giab008. <https://doi.org/10.1093/gigascience/giab008> <https://www.htslib.org/>
[^3]: <http://broadinstitute.github.io/picard>