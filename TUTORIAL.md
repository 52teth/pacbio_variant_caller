# Calling SVs in yeast

## Configure distributed environment

SMRT-SV uses DRMAA to submit jobs to a grid-engine-style cluster. To enable the `--distribute` option of SMRT SV, add the following line to your `.bash_profile` with the correct path to the DRMAA library for your cluster.

```bash
export DRMAA_LIBRARY_PATH=/opt/uge/lib/lx-amd64/libdrmaa.so.1.0
```

Alternately, provide the path to your DRMAA library with the SMRT-SV
`--drmaalib` option.

Additionally, you may need to configure resource requirements depending on your
cluster and PacBio data. Use the `--cluster_config` option when running SMRT-SV
to pass a JSON file that specifies [Snakemake-style cluster
parameters](https://bitbucket.org/snakemake/snakemake/wiki/Documentation#markdown-header-cluster-configuration). An
example configuration used to run SMRT-SV with human genomes on the Eichler lab
cluster is provided in this repository in the file `cluster.eichler.json`.

## Download PacBio reads

```bash
# List of AWS-hosted files from PacBio including raw reads and an HGAP assembly.
wget https://gist.githubusercontent.com/pb-jchin/6359919/raw/9c172c7ff7cbc0193ce89e715215ce912f3f30e6/gistfile1.txt

# Keep only .xml, .bas.h5, and .bax.h5 files.
sed '/fasta/d;/fastq/d;/celera/d;/HGAP/d' gistfile1.txt > gistfile1.keep.txt

# Download data into a raw reads directory.
mkdir -p raw_reads
cd raw_reads
for f in `cat ../gistfile1.keep.txt`; do wget --force-directories $f; done

# Create a list of reads for analysis.
cd ..
find ./raw_reads -name "*.bax.h5" -exec readlink -f {} \; > reads.fofn
```

## Prepare the reference assembly

Download the reference assembly (sacCer3) from UCSC.

```bash
mkdir -p reference
cd reference
wget ftp://hgdownload.cse.ucsc.edu/goldenPath/sacCer3/bigZips/chromFa.tar.gz
```

Unpack the reference tarball and concatenate individual chromosome files into a
single reference FASTA file.

```bash
tar zxvf chromFa.tar.gz
cat *.fa > sacCer3.fasta
rm -f *.fa *.gz
cd ..
```

Prepare the reference sequence for alignment with PacBio reads. This step
produces suffix array and ctab files used by BLASR to speed up alignments.

```bash
smrtsv.py index reference/sacCer3.fasta
```

## Align reads to the reference

Align reads to the reference with BLASR.

```bash
smrtsv.py align reference/sacCer3.fasta reads.fofn
```

## Find signatures of variants in raw reads

Find candidate regions to search for SVs based on SV signatures.

```bash
smrtsv.py detect reference/sacCer3.fasta alignments.fofn candidates.bed
```

## Assemble regions

Assemble local regions of the genome that have SV signatures or tile across the
genome.

```bash
smrtsv.py assemble reference/sacCer3.fasta reads.fofn alignments.fofn candidates.bed local_assembly_alignments.bam
```

## Call variants

Call variants by aligning tiled local assemblies back to the
reference. Optionally, specify the sample name for annotation of the final VCF
file and a species name (common or scientific as supported by
[RepeatMasker](http://www.repeatmasker.org/)) for repeat masking of structural
variants.

```bash
smrtsv.py call reference/sacCer3.fasta alignments.fofn local_assembly_alignments.bam variants.vcf --sample UCSF_Yeast9464 --species yeast
```
