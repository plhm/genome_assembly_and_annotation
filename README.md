
Step 1: Preliminary QC & Quality Trimming
======
Preliminary QC of your sequences can be completed by applying [`fastqc`](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/) to your fastq sequence files. [`fastqc`](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/) is a popular tool that "aims to provide a simple way to do some quality control checks on raw sequence data coming from high throughput sequencing pipelines." Carefully inspect the `html` output from [`fastqc`](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/) because this is the main way that you are going to make sure that nothing is seriously wrong with your data before you delve into the time consuming series of analyses discussed below.
```
fastqc -k 6 *
```

If your sequences look OK after preliminary QC, its time to get your sequences ready for downstream analyses by trimming adaptors and eliminating low quality sequences and basecalls. Here we do this operation on the reads generated from the short insert library. We are going to do this by using the function `cutadapt` to (1) trim Illumina adapter sequences, (2) discard reads <75 bp in length and (3) perform gentle trimming of low quality basecalls. This process should take around 12 hours to complete for a raw sequence file containing around 500 million 100-150 bp reads.
```
#PBS -N cutadapt_short.sh
#PBS -l nodes=1:ppn=1:avx,mem=16000m,walltime=48:00:00
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -d /scratch/glor_lab/rich/distichus_genome/Short_Insert
#PBS -j oe
#PBS -o cutadapterror_short

fastqc -k 6 /scratch/a499a400/anolis/deliv_Glor061715_raw/raw_fastq/Anolis_Genome_R1.fastq
fastqc -k 6 /scratch/a499a400/anolis/deliv_Glor061715_raw/raw_fastq/Anolis_Genome_R2.fastq
cutadapt -a AGATCGGAAGAGC -A AGATCGGAAGAGC -q 5 -m 25 -o Short_trimmed_R1.fastq.gz -p Short_trimmed_R2.fastq.gz /scratch/a499a400/anolis/deliv_Glor061715_raw/raw_fastq/Anolis_Genome_R1.fastq /scratch/a499a400/anolis/deliv_Glor061715_raw/raw_fastq/Anolis_Genome_R2.fastq > cutadapt.log
```
After trimming is complete, use `fastqc` on the resulting files to check that adapters have been trimmed and that the newly generated `fastq` files look good. 

Step 2: Correct Sequencing Errors
======
Although Illumina is generally regarded as a relatively error-free sequencing method, your sequences will still include plenty of errors, most of which will be substitution errors. One way to fix these types of sequencing erros is to use a k-mer spectrum error correction framework such as EULER or Quake ([Kelley et al. 2010](http://genomebiology.com/2010/11/11/R116)). The basic idea of these approaches involves identification of particularly low frequency k-mers that are likely the result of sequencing errors. The [publication introducing Quake](http://genomebiology.com/2010/11/11/R116) has a nice introduction to the underlying algorithms. More recently Trowel has been introduced and has the advantage of incorporating information on quality scores.

Step 2a: Quake
-----
We use [Quake](http://www.cbcb.umd.edu/software/quake/index.html) here. Quake uses Jellyfish for k-mer counting. Running Quake is fairly straightforward, with simple instructions available via the [program's online manual](http://www.cbcb.umd.edu/software/quake/manual.html). If you are uncertain of what k-mer size to use, the [Quake FAQ has a suggestion](http://www.cbcb.umd.edu/software/quake/faq.html). Quake can't unzip zipped sequence files on the fly; in the script below, I attempt to do this procedure without taking up too much space by unzipping on the compute node and compressing before copying back to the scratch directory. 
```
#PBS -N quake_short
#PBS -q default -l nodes=1:ppn=24:avx,mem=50000m,walltime=148:00:00,file=300gb
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -d /scratch/glor_lab/rich/distichus_genome/Quake
#PBS -j oe
#PBS -o quake_short_error

work_dir=$(mktemp -d)
mkdir $work_dir
gzip -dc /scratch/glor_lab/rich/distichus_genome/Short_Insert/Short_trimmed_R* $work_dir
quake.py -f $work_dir/Short_trimmed_R2.fastq $work_dir/Short_trimmed_R1.fastq -k 17 -p 24
gzip $work_dir/*fastq
rm $work_dir/Short_trimmed_R1.fastq.gz
rm $work_dir/Short_trimmed_R2.fastq.gz
mv $work_dir/* /scratch/glor_lab/rich/distichus_genome/Quake
rm -rf $work_dir
```
Step 2b: Trowel
-----
For more on Trowel, see the publication responsible for this tool ([Lim et al. 2014](http://dx.doi.org/10.1093/bioinformatics/btu513)).
```
#PBS -N trowel_short
#PBS -q bigm -l nodes=1:ppn=24:avx,mem=512000m,walltime=148:00:00,file=300gb
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -j oe
#PBS -d /scratch/glor_lab/rich/distichus_genome/Trowel
#PBS -o trowel_short_error

work_dir=$(mktemp -d)
mkdir $work_dir
cp /scratch/glor_lab/rich/distichus_genome/Short_Insert/Short_trimmed* $work_dir
gunzip -c $work_dir/Short_trimmed_R*
trowel -k 17 -t 24 -f files_short.txt
gzip $work_dir/*fastq
rm $work_dir/Short_trimmed_R1.fastq.gz
rm $work_dir/Short_trimmed_R2.fastq.gz
mv $work_dir/* /scratch/glor_lab/rich/distichus_genome/Trowel
rm -rf $work_dir
```
Step 3:*De novo* Assembly of Small Insert Library with DISCOVAR
======
```
#!/bin/bash
#SBATCH -N 1
#SBATCH --ntasks-per-node=16
#SBATCH --mem=1450G
#SBATCH --partition=large-shared
#SBATCH -A kan110
#SBATCH --no-requeue
#SBATCH -t 2880 

export MALLOC_PER_THREAD=1
/home/aalexand/bin/discovardenovo-52488/bin/DiscovarDeNovo READS=/oasis/scratch/comet/aalexand/temp_project/combined_Anolis_Genome_R1.fastq,/oasis/scratch/comet/aalexand/temp_project/combined_Anolis_Genome_R2.fastq OUT_DIR=/oasis/scratch/comet/aalexand/temp_project/17Feb2016 NUM_THREADS=15 MAX_MEM_GB=1000 MEMORY_CHECK=True
```

Step 4: Genome Completeness, Coverage and Size
======

Step 4a: Mapping Reads to Genome
------
Mapping short reads back to the final genome is a basic way of assessing overall genome coverage. Here we map the short reads from the short insert library to the final Dovetail assembly.
```
#PBS -N mapping_dovetail_short
#PBS -q default -l nodes=1:ppn=24:avx,mem=50000m,walltime=168:00:00
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -d /scratch/glor_lab/rich/distichus_genome
#PBS -j oe
#PBS -o mapping_dovetail_short_error

bowtie2-build /scratch/a499a400/anolis/dovetail/trunk_anole_19Jun2016_xkeD9.fasta Dovetail_bowtie_DB.fasta
bowtie2 --local --no-unal -x Dovetail_bowtie_DB.fasta -q -1 /scratch/glor_lab/rich/distichus_genome/Short_Insert/Short_trimmed_R1.fastq.gz -2 /scratch/glor_lab/rich/distichus_genome/Short_Insert/Short_trimmed_R2.fastq.gz | samtools view -Sb - | samtools sort -no - - > bowtie2.dovetail.nameSorted.bam
SAM_nameSorted_to_uniq_count_stats.pl bowtie2.dovetail.nameSorted.bam > bowtie_dovetail_mapping_summary

```
To get a summary of the mapping results, we can use a perl script from `Trinity`.
```
SAM_nameSorted_to_uniq_count_stats.pl bowtie2.nameSorted.bam

#read_type      count   pct
proper_pairs    30446172        94.39
improper_pairs  928085  2.88
left_only       699683  2.17
right_only      181046  0.56

Total aligned rnaseq fragments: 32254986
```
Step 4b: 17-mer Estimate of Genome Coverage
------
We going to use methods first used in the Panda genome paper ([Ruiqiang et al. 2010](http://dx.doi.org/10.1038/nature08696)). We will begin by using the k-mer counting software `jellyfish` to generate a distribution of k-mer coverage. This operation shouldn't take more than 12 hours when conducted on our short insert library including 183M paired reads.

```
#PBS -N jellyfish_short
#PBS -q bigm
#PBS -l nodes=1:ppn=16:avx,mem=20000m,walltime=12:00:00
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -j oe
#PBS -d /scratch/glor_lab/rich/distichus_genome/Jellyfish
#PBS -o jellyfish_short_error

work_dir=$(mktemp -d)
cat /scratch/glor_lab/rich/distichus_genome/Short_Insert/Short_trimmed_R1.fastq /scratch/glor_lab/rich/distichus_genome/Short_Insert/Short_trimmed_R2.fastq > $work_dir/Short_trimmed_merged.fastq
jellyfish count -m 17 -o fastq.counts -C $work_dir/Short_trimmed_merged.fastq -s 10000000000 -U 500 -t 16
rm $work_dir/Short_trimmed_merged.fastq
mv $work_dir/* /scratch/glor_lab/rich/distichus_genome/Jellyfish/
rm -rf $work_dir
```

We can then make a histogram from this data as follows.
```
jellyplot.pl fastq.counts > fastq.counts.histo
```
The resulting histogram file can then be uploaded and evaluated by [GenomeScope](http://qb.cshl.edu/genomescope/), an online tool capable of providing assembly free estimates of genome size, coverage, heterozygosity and other statistics.

Step 5: Assess Genome Completeness by Identifying Coverage of Expected Proteins
======
One important benchmark for genome quality involves asking how many of the proteins that are expected to occur in our organism based on prior genomic studies are present in our assembled genome. 

Step 5a: Benchmarking Universal Single-Copy Orthologs (BUSCO)
------
"BUSCO v2 provides quantitative measures for the assessment of genome assembly, gene set, and transcriptome completeness, based on evolutionarily-informed expectations of gene content from near-universal single-copy orthologs selected from OrthoDB v9." For more about BUSCO visit the [project's website](http://busco.ezlab.org/) or read the paper reporting the method ([Simão et al. 2015](http://dx.doi.org/10.1093/bioinformatics/btv351)).
```
#PBS -N busco_dovetail.sh
#PBS -l nodes=1:ppn=24:avx,mem=64000m,walltime=72:00:00
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -d /scratch/glor_lab/rich/distichus_genome/
#PBS -j oe
#PBS -o busco_dovetail_error

BUSCO.py -o busco_dovetail -i /scratch/a499a400/anolis/dovetail/trunk_anole_19Jun2016_xkeD9.fasta -l /scratch/glor_lab/rich/distichus_genome_RNAseq/vertebrata_odb9 -m geno
```
After running this operation, you should find a simple summary of your data (`short_summary_busco_dewlap.txt`) in a folder called `run_busco_name` that will tell you how many complete BUSCOs were recovered in your dataset.
```
       XX
```

Step 5b: Assess Full-length Transcripts Relative to Reference Via BLAST+
------
We can also ask how many of the proteins in the Anolis carolinensis proteome are present in our genome.
```
#PBS -N blast_anole_dovetail.sh
#PBS -l nodes=1:ppn=24:avx,mem=200000m,walltime=140:00:00
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -d /scratch/glor_lab/rich/distichus_genome/Anole_BLAST
#PBS -j oe
#PBS -o blast_anole_dovetail_error

makeblastdb -in ASU_Acar_v2.1.prot.fa -dbtype prot
blastx -query /scratch/a499a400/anolis/dovetail/trunk_anole_19Jun2016_xkeD9.fasta -db ASU_Acar_v2.1.prot.fa -out blastx.outfmt6 -evalue 1e-20 -num_threads 24 -max_target_seqs 1 -outfmt 6
analyze_blastPlus_topHit_coverage.pl blastx.outfmt6 /scratch/a499a400/anolis/dovetail/trunk_anole_19Jun2016_xkeD9.fasta ASU_Acar_v2.1.prot.fa > analyze_blastPlus_topHit_coverage.log
/tools/cluster/6.2/trinityrnaseq/2.2.0/util/misc/blast_outfmt6_group_segments.pl blastx.outfmt6 /scratch/a499a400/anolis/dovetail/trunk_anole_19Jun2016_xkeD9.fasta ASU_Acar_v2.1.prot.fa > blast.outfmt6.grouped
/tools/cluster/6.2/trinityrnaseq/2.2.0/util/misc/blast_outfmt6_group_segments.tophit_coverage.pl blast.outfmt6.grouped > analyze_groupsegments_topHit_coverage.log

```
Step 6: Assess Repetitive Content of Genome
======
We are going to use the function `RepeatMasker` to generate a basic assessment of repetitive content of our genome. We will use the basic function `RepeatMasker` to identify repetitive content from the RepeatMasker database. We will then use the function `RepeatModeler` to generate a library of repetitive elements for our species of interest. Similar processes are also integrated in Maker2.

Step 6a: RepeatMasker
------
```
#PBS -N repeatmasker_dovetail.sh
#PBS -q default -l nodes=1:ppn=24:avx,mem=64000m,walltime=148:00:00,file=200gb
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -d /scratch/glor_lab/rich/distichus_genome/RepeatMasker
#PBS -j oe
#PBS -o repeatmasker_dovetail_error

work_dir=$(mktemp -d)
mkdir $work_dir
cp /scratch/a499a400/anolis/dovetail/trunk_anole_19Jun2016_xkeD9.fasta $work_dir
RepeatMasker -e ncbi -species vertebrates $work_dir/trunk_anole_19Jun2016_xkeD9.fasta >> RepeatMasker_Dovetail.log
mv $work_dir/ /scratch/glor_lab/rich/distichus_genome/RepeatMasker
rm -rf $work_dir
```

Step 6b: RepeatModeler
------
First run RepeatModeler
```
#PBS -N repeatmodeler_dovetail.sh
#PBS -l nodes=1:ppn=24:avx,mem=64000m,walltime=148:00:00
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -d /scratch/glor_lab/rich/distichus_genome/RepeatModeler
#PBS -j oe
#PBS -o repeatmodeler_dovetail_error

BuildDatabase -name distichus_dovetail -engine ncbi /scratch/a499a400/anolis/dovetail/trunk_anole_19Jun2016_xkeD9.fasta
RepeatModeler -engine ncbi -pa 24 -database distichus_dovetail
```

Then run RepeatMasker with RepeatModeler output.
```
#PBS -N repeatmasker_modeler_dovetail.sh
#PBS -q default -l nodes=1:ppn=24:avx,mem=64000m,walltime=148:00:00,file=200gb
#PBS -M glor@ku.edu
#PBS -m abe
#PBS -d /scratch/glor_lab/rich/distichus_genome/RepeatModeler
#PBS -j oe
#PBS -o repeatmasker_modelerdovetail_error

work_dir=$(mktemp -d)
mkdir $work_dir
cp /scratch/a499a400/anolis/dovetail/trunk_anole_19Jun2016_xkeD9.fasta $work_dir
RepeatMasker -lib /scratch/glor_lab/rich/distichus_genome/RepeatModeler/RM_101306.FriDec91754042016/consensi.fa.classified $work_dir/trunk_anole_19Jun2016_xkeD9.fasta >> RepeatMasker_Modeler_Dovetail.log
mv $work_dir/* /scratch/glor_lab/rich/distichus_genome/RepeatModeler/
rm -rf $work_dir
```

Step 7: *Ab initio* Gene Prediction With AUGUSTUS
======
We are going to run a preliminary ab initio assembly using [AUGUSTUS](http://dx.doi.org/10.1093/bioinformatics/btg1080).


Step 8: Genome Annotation in Maker2
======
Give Maker: assembled genome fasta file, transcripts, A. carolinensis proteome, repeat library from distichus, reference repeat library from vertebrates, softmasking
Use BUSCO output as initial parameters for Augustus/Maker. May need to use long flag with busco
Conflict over whether using bowtie output or raw transcripts does better in Maker
Check GMOD/Maker Wiki
Augustus does not require iterative training
SNAP may need to run multiple times
Run at first with parameters set to 1
Run again with re-trained SNAP file
AED (Annotation edit distance) smaller value = better evidence
