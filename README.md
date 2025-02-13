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
singularity exec /lustre/isaac/scratch/jtorre28/singularity_containers/busco:v5.7.1_cv1.sif busco -i /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.fasta --lineage clostridia_odb10 -m geno -o /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.clos.busco -f
```


