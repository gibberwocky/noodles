# Single-cell universal preprocessing workflow #

The [cellatlas](https://github.com/cellatlas/cellatlas) repo is a [universal preprocessing workflow](https://www.biorxiv.org/content/10.1101/2023.09.14.543267v1.full) for single-cell data. It leverages the [seqspec](https://academic.oup.com/bioinformatics/article/40/4/btae168/7641535?login=false) assay specification. The initial seqspec release contains specifications for 49 assays, inclusing [SPLIT-seq](Single-cell profiling of the developing mouse brain and spinal cord with split-pool barcoding).

## Initiate conda environment ##

```bash
# Create conda environment
module load mamba
mamba create --name cellatlas python=3.7
mamba activate cellatlas
```

## Install dependencies ##

```bash
cd ~/software

# Install `tree` to view files
git clone https://github.com/adsr/tree.git
cd tree
make -j16 && make install

# Install `jq, a command-line tool for extracting key value pairs from JSON files 
wget --quiet --show-progress https://github.com/stedolan/jq/releases/download/jq-1.7.1/jq-linux64
chmod +x jq-linux64

# Install gget (https://github.com/pachterlab/gget) for efficientey querying of genomic dbs
#wget --quiet --show-progress https://github.com/pachterlab/gget/archive/refs/tags/v0.29.0.tar.gz
mamba install gget

# Install kb-python (https://github.com/pachterlab/kb_python) wrapper for kallisto and bustools
mamba install kb-python

# Install git pip
mamba install git pip
```

## Install cellatlas and seqspec

```bash
# Revert seqspec to an earlier version
pip install --quiet git+https://github.com/cellatlas/cellatlas.git > /dev/null
#yes | pip uninstall --quiet seqspec
# cellatlas developed with an in-between release of seqspec
pip install --quiet git+https://github.com/pachterlab/seqspec.git@9471a317f524c289ee6582c1889cdeac0c5396b2
```

## Export environment

```bash
mamba env export > cellatlas.yml

# In theory, env can be recreated with
# mamba env create -f cellatlas.yml
```


## SPLIT-seq example ##
 
```bash
mkdir ~/cellatlas-example && cd ~/cellatlas-example
mv ~/software/cellatlas/examples/rna-splitseq/* .
gunzip barcode*
seqspec print spec.yaml
```

The next lines in the example do not appear to function as intended:

```bash
FA=echo $(~/software/jq-linux64 -r '.mus_musculus.genome_dna.ftp' ref.json)
FA=FA[0]
GTF=echo $(~/software/jq-linux64 -r '.mus_musculus.annotation_gtf.ftp' ref.json)
GTF=GTF[0]
```

But it is clear that ```FA``` and ```GTF``` simply point to the reference fasta and GTF annotation file, respectively. 
These can simply be downloaded from the paths mentioned. It is worthwhile checking that AnnotationHub supports the release
listed, as it may be preferable to use an earlier release if it doesn't.

```bash
#wget http://ftp.ensembl.org/pub/release-113/fasta/mus_musculus/dna/Mus_musculus.GRCm39.dna.primary_assembly.fa.gz
#wget http://ftp.ensembl.org/pub/release-113/gtf/mus_musculus/Mus_musculus.GRCm39.113.gtf.gz
wget http://ftp.ensembl.org/pub/release-109/fasta/mus_musculus/dna/Mus_musculus.GRCm39.dna.primary_assembly.fa.gz
wget http://ftp.ensembl.org/pub/release-109/gtf/mus_musculus/Mus_musculus.GRCm39.109.gtf.gz
```

Run cell atlas, specifying the output directory ```-o out```, modality ```-m rna```, 
seqspec file ```-s spec.yaml```, ```fasta```, ```gtf```, and ```fastq``` input files.

```bash
cellatlas build -o out -m rna -s spec.yaml \
  -fa Mus_musculus.GRCm39.dna.primary_assembly.fa.gz \
  -g Mus_musculus.GRCm39.109.gtf.gz \
  fastqs/R1.fastq.gz fastqs/R2.fastq.gz
```

To view the commands generated by cellatlas written to ```out/cellatlas_info.json```:

```bash
echo $(~/software/jq-linux64  -r '.commands[] | values[] | join("\n")' out/cellatlas_info.json)
```

To append the commands to a slurm template script for running via ```sbatch```:

```bash
cp ~/templates/cellatlas.sh out
~/software/jq-linux64  -r '.commands[] | values[] | join("\n")' out/cellatlas_info.json >> out/cellatlas.sh
chmod +x out/cellatlas.sh
sbatch out/cellatlas.sh
```


### To check RAM usage of running job

```bash
# -j is slurm job id
sstat -j 2145401 --format="MaxRSS"
```

### To check job efficiency (i.e. CPU and RAM usage) post-completion

```bash
# number is slurm job id
seff 2141046
```

## Read data into R

We're interested in the ```h5ad``` file, with a view to importing it into R on laptop as a ```SingleCellExperiment```.

#sreport cluster UserUtilizationByAccount users=s14dw4 start=2000-01-01 format=Accounts%20,Cluster%16,Login,Used

```bash
# On own laptop, copy unfiltered counts matrix from Maxwell
scp s14dw4@maxlogin1:/uoa/home/s14dw4/cellatlas-example/out/counts_unfiltered/adata.h5ad ~/Documents/cellatlas
```