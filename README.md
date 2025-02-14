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
> file prefix name set to "redo" as the first run crashed 

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

> Creates the initial blobtools directory using the fasta file
```
blobtools create --fasta /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.fasta Ur_e
```

> Adds the read alignment data to the blob directory
```
blobtools add --cov /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.bam Ur_e
```

> Adds the BUSCO results to the blob directory
```
blobtools add --busco /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.busco/run_arthropoda_obd10/full_table.tsv Ur_e
```

> Creates interactive browser session with blobtool directory to obtain and adjust plots as well as obtain a list of the taxonomy assignments.
```
blobtools view --local --port 8002-8009 Ur_e
```

Once taxonomy list is obtained though blobtools a list of know contmainants can be made 
(Non-arthropoda taxonomy assignments)

This list of contig names (ids.txt) can be used to remove the contaminaants with an awk command

```
awk 'BEGIN{while((getline<"ids.txt")>0)l[">"$1]=1}/^>/{f=!l[$1]}f' lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.fasta > redo.contigs.filt.fa
```

### Post-purging Quality asseement with BUSCO singularity image

BUSCO is re-ran on the assmebly after removing to contaminants to prevent over-purging the assembly. In this casce the busco score remained exactly the same.

```
singularity exec /lustre/isaac/scratch/jtorre28/singularity_containers/busco:v5.7.1_cv1.sif busco -i /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.filt.fa --lineage arthropoda_odb10 -m geno -o /lustre/isaac/scratch/jtorre28/Ue_open/busco.redo.contigs.filt.out -f
```

# Assembly of nanopore reads with next denovo. 

To increase the continuity of the assmebly the same reads were also assembled with Next-denovo as they usually have longer contigs at the cost of a less complete assembly

> The following run file was used as the parameters based on the computational specifactions of the node used.

```
[General]
job_type = local
job_prefix = nextDenovo
task = all # 'all', 'correct', 'assemble'
rewrite = yes # yes/no
deltmp = yes
rerun = 3
parallel_jobs = 3
input_type = raw
read_type = ont
input_fofn = ./input.fofn
workdir = ./01_rundir

[correct_option]
read_cutoff = 1k
genome_size = 450m
pa_correction = 2
sort_options = -m 1g -t 2
minimap2_options_raw =  -t 8
correction_options = -p 15

[assemble_option]
minimap2_options_cns =  -t 8
nextgraph_options = -a 1
```

The input.fofn file refenced has the list of input files; In this case the files were already concatenated into one file for canu 

```
/lustre/isaac/scratch/jtorre28/minimap2/Ue_nano.fastq
```
After the assembly, both Canu and next-denovo assemblies were combined with Quickmerge to maintain the same completeness as canu while increasing the contig legths withnaxt-denovo. 

> The canu assmebly was used as the refrence meaning that the unmerged contigs from this assembly would be kept so the completness wouldremain the same. 

```
merge_wrapper.py -p ue-sealed  /lustre/isaac/scratch/jtorre28/Ue_open/redo.contigs.filt.fa /lustre/isaac/scratch/jtorre28/next-denovo/NextDenovo/ue/01_rundir/03.ctg_graph/nd.asm.v2.fasta
```

BUSCO was re-run on the merged assembly to ensure no loss in completeness

```
singularity exec /lustre/isaac/scratch/jtorre28/singularity_containers/busco:v5.7.1_cv1.sif busco -i /lustre/isaac/scratch/jtorre28/Ue-patched/ue-sealed.filt.fa -l hemiptera_odb10 -c 24 -m geno -o busco.ue.sealed.out -f
```

> Another blob directory was made with this new assembly to genreate the final snail and blob plot to ensure no contaminants were introduced


## Gene prediction of assembly with Helixer

```
singularity exec --nv --env 'RUST_BACKTRACE=full' /lustre/isaac/scratch/jtorre28/singularity_containers/helixer-docker_helixer_v0.3.3_cuda_11.8.0-cudnn8.sif Helixer.py --fasta-path  /lustre/isaac/scratch/jtorre28/Ue-patched/ue-sealed.filt.fa --gff-output-path /lustre/isaac/scratch/jtorre28/Ue-patched/ue-sealed.gff --lineage invertebrate --species Uroleucon_erigeronense --temporary-dir /lustre/isaac/scratch/jtorre28/singularity_containers/tmp
```

## functional annotation with entap
transcript files were made using the reulting gff file output from helixer







