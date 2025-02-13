# Uroleucon-eupatorifoliae-assembly-to-annotation

Sanatize basecalled reads ( removes formatting/parsing errors )

```
 seqkit sana /lustre/isaac/scratch/jtorre28/minimap2/Ue_nano.fastq -o /lustre/isaac/scratch/jtorre28/minimap2/Ue-reads.sana.fastq
```

## assembly length prediction with Jellyfish and genomescope

> First counting all 20-mers
Note that the sanatized reads are used as input

```
jellyfish count -m 20 -s 100M -t 45 -C /lustre/isaac/scratch/jtorre28/minimap2/Ue-reads.sana.fastq
```

> Use the k-mer counts to make a histogram
defalt output of previous command is imaginatively named mer_counts.jf

```
jellyfish histo -t 35 mer_counts.jf > A-sol-cor_reads20.histo
```
> Upload histogram file to
```
http://genomescope.org/
```
this yields a histogram figure as well as coverage and length predictions

## Canu assembly

Using the length pedictions from genomscope

```
canu -p redo -d /lustre/isaac/scratch/jtorre28/Ue_open genomeSize=436m utgReAlign=true overlapper=mhap corMhapFilterThreshold=0.0000000002 corMhapOptions="--threshold 0.80 --num-hashes 512 --num-min-matches 3"  -nanopore /lustre/isaac/scratch/jtorre28/minimap2/Ue_nano.fastq usegrid=false gridOptions="--partition=campus --qos=campus --time=24:00:00 --account ACF-UTK0011"
```
### Quality asseement with BUSCO singularity image

> initial assembly was scanned with BUSCO to determine the initial completeness before purging contamination.

```
singularity exec /lustre/isaac/scratch/jtorre28/singularity_containers/busco:v5.7.1_cv1.sif busco -i /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.fasta --lineage arthropoda_odb10 -m geno -o /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.busco -f
```

## Preparing taxonomy assignments
Diamond blastx was used to assign taxonomy alignments to contigs using a clustered NR database

```
        diamond blastx --db /lustre/isaac/scratch/jtorre28/database/cluster-nr/nr_tax.dmnd \
       -q /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.fasta \
       --outfmt 6 qseqid staxids bitscore qseqid sseqid pident length mismatch gapopen qstart qend sstart send evalue bitscore \
       --sensitive \
       --max-target-seqs 1 \
       --evalue 1e-25 \
       --threads 48 \
       -F 15 \
       --out /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.dmnd.out
```

## Alignment of nanopore reads to assembly
The original nanopore reads were mapped back to the assembly to screen for contaminants

```
minimap2 -ax map-ont \
         -t 40 /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.fasta \
         /lustre/isaac/scratch/jtorre28/minimap2/Ue-reads.sana.fastq \
 | samtools sort -@40 -O BAM -o /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.bam -
```
## Visualisation of coverage, taxonomy assignment, and BUSCO score with Blobtools visualizer
Blobtools was used to generate a snail plot, blob plot, and a TSV file containing the full predicted lineage of each contig assmbled 

> Create the initial blobtools directory using the fasta file
```
blobtools create --fasta /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.fasta Ur_e
```

> Add the read alignment data to the blob directory
```
blobtools add --cov /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.bam Ur_e
```

> Add the BUSCO results to the blob directory
```
blobtools add --busco /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.busco/run_arthropoda_obd10/full_table.tsv Ur_e
```



