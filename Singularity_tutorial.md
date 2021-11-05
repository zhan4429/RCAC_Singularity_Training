## What is container?  
- A **container** is an abstraction for a set of technologies that aim to solve the problem of how to get software to run reliably when moved from one computing environment to another.  
- A container **image** is simply a file (or collection of files ) saved on disk that stores everything you need to run a target application or applications.  

## Docker  
- The concept of containers emerged in 1970s, but they were not well known until the emergence of Docker containers in 2013.  
- Docker is an open source platform for building, deploying, and managing containerized applications.   
- `Some concerns about the security of Docker containers in HPC`: Docker gives superuser privileges, but we do not want users to have full, unrestricted admin/ root access on production systems.  

## Singularity  
Singularity was developed in 2015 as an open-source project by researchers at Lawrence Berkeley National Laboratory led by Gregory Kurtzer.   
Singularity is emerging as the containerization framework of choice in HPC environments.   
1. Enable researchers to package entire scientific workflows, libraries, and even data.
2. Users do not need to ask their system admin (e.g., RCAC) to install software for them.
3. Can use docker images.
4. Secure! 
5. Does not require root privileges.

## Singularity basics 
Detailed singularity user guide is available at: [sylabs.io/guides/3.8/user-guide](https://sylabs.io/guides/3.8/user-guide/).  
The main singularity command:  
```
$ singularity [options] <subcommand> [subcommand options …]
```


## Practice singularity in RCAC HPC clusters  
### Login clusters  
```
$ ssh USRID@CLUSTER.rcac.purdue.edu # You can login any Purdue cluster you have access

$ cd $RCAC_SCRATCH # We will practice in our scratch directory
```


### singularity pull  
Download or build a container from a given URI. 
```
$ singularity pull [output file] <URI>
```

Let's pull a bowtie2 container from [Docker Hub](https://hub.docker.com/r/biocontainers/bowtie2/tags). [Bowtie 2](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml) is an ultrafast and memory-efficient tool for aligning sequencing reads to long reference sequences.  
```
$ singularity pull bowtie2_v2_4_1.sif  docker://biocontainers/bowtie2:v2.4.1_cv1
```
We can see that the container image file `bowtie2_v2_4_1.sif` was pull to our current directory. In the next section, we willuse this `bowtie2` container to align our read sequences to the reference genomes.  


### singularity shell  

Let's go inside the pulled `bowtie2` container.  

```
$ singularity shell bowtie2_v2_4_1.sif 
Singularity>/etc/*release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.6 LTS"
NAME="Ubuntu"
VERSION="16.04.6 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.6 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
VERSION_CODENAME=xenial
UBUNTU_CODENAME=xenial

Singularity> which bowtie2
/usr/local/bin/bowtie2
```

### singularity exec
With the `bowtie2` container, we can do some real research. In the `Inputs` folder, I put two fastq files (input_1.fastq and input_2.fastq)from pair-end sequencing of the bacteria *Escherichia coli*. In addtion, the `Inputs` folder also has the a file `Ecoli_K12.fasta` containing the reference *E. coli* K12 genome. Let's align the two fastq files against the reference `Ecoli_K12.fasta` with our `Bowtie2` container. The details about `Bowtie2` usage is available [here](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml). 

```
## build the index of the reference genome 
$ singularity exec bowtie2_v2_4_1.sif bowtie2-build Inputs/Ecoli_K12.fasta Inputs/Ecoli_K12
## Align fastq reads to reference genome
$ singularity exec bowtie2_v2_4_1.sif bowtie2 -x Inputs/Ecoli_K12 -1 Inputs/input_1.fastq -2 Inputs/input_2.fastq -S Ecoli_out.sam

perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").
10000 reads; of these:
  10000 (100.00%) were paired; of these:
    191 (1.91%) aligned concordantly 0 times
    9419 (94.19%) aligned concordantly exactly 1 time
    390 (3.90%) aligned concordantly >1 times
    ----
    191 pairs aligned concordantly 0 times; of these:
      145 (75.92%) aligned discordantly 1 time
    ----
    46 pairs aligned 0 times concordantly or discordantly; of these:
      92 mates make up the pairs; of these:
        42 (45.65%) aligned 0 times
        24 (26.09%) aligned exactly 1 time
        26 (28.26%) aligned >1 times
99.79% overall alignment rate
```
Some containers may give us warnings, however, in most cases, we can just ignore them.  




### singularity build  
To build a singularity container, we need to use a computer with elevated privileges, then copy or pull to cluster. To build the container in RCAC clusters, we can build remotely using the [Sylabs Remote Builder](https://cloud.sylabs.io/builder).  
To remotely build an image using singularity, go through the following steps:  
1. Go to: https://cloud.sylabs.io/, and generate a Sylabs account. 
2. Create a new `Access Tokens`, and copy it to clipboard.
3. SSH login to our clusters, and run `singularity remote login` in terminal and paste the access token at the prompt.
4. Then you can remotely build your own singularity image in the cluster.  

Example: build our own prokka container  
[MACS](https://github.com/macs3-project/MACS) is a widely used tool for identifying transcription factor binding sites for ChIP-Seq or ATAC-Seq analysis. Here we will demonstrate how to build our own macs tool (version 2.2.7.1) with the definition file shared by [NIH-HPC staff](https://github.com/NIH-HPC/singularity-def-files/tree/master/high-throughput-sequencing/macs/2.2.7.1). 

The prokka defination file 
```
BootStrap: docker
From: debian:buster-slim

######################################################################
%post
######################################################################
apt-get -y update
apt-get install -y curl wget nano bzip2 less

# miniconda
mkdir -p /opt
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -f -b -p /opt/conda
. /opt/conda/etc/profile.d/conda.sh
conda activate base
conda update --yes --all
conda config --add channels bioconda
conda config --add channels conda-forge
conda create -n prokka prokka==1.14.6


# create bind points for NIH HPC environment
mkdir /gpfs /spin1 /data /scratch /fdb /lscratch /vf
for i in $(seq 1 20); do ln -s /gpfs/gsfs$i /gs$i; done

# clean up
apt-get clean
conda clean --yes --all

######################################################################
%environment
######################################################################
export LC_ALL=C
export PATH=/opt/conda/envs/prokka/bin:$PATH

```

Let's build our first container.  
```
$ singularity remote login
##Paste the access token at the prompt.  
$ singularity build --remote prokka.sif Inputs/prokka.def 
```

This will take some time, maybe more than 10 mins. After it finishes, you can see information simialar to below in the terminal:  
```bash
Container URL: https://cloud.sylabs.io/library/zhan4429/remote-builds/rb-618402da4f908fd1f584814c 
INFO:    Build complete: prokka.sif
```

Congratulations for your first self-built container :smiley: :smiley: :+1: :+1:  

I cannot wait to run my first prokka container. In the `Inputs` directory, I put a bacteria genome belonging to Escherichia coli K12. We can use prokka to annotate the genome.  
```
$ singularity exec prokka.sif prokka --outdir prokka_EcoliK12  --prefix K12 Inputs/Ecoli_K12.fasta
```

Since this is only a small bacterial genome, prokka will finish within several minutes. 
```
[12:20:35] Output files:
[12:20:35] prokka_EcoliK12/K12.gbk
[12:20:35] prokka_EcoliK12/K12.ffn
[12:20:35] prokka_EcoliK12/K12.faa
[12:20:35] prokka_EcoliK12/K12.sqn
[12:20:35] prokka_EcoliK12/K12.gff
[12:20:35] prokka_EcoliK12/K12.fna
[12:20:35] prokka_EcoliK12/K12.fsa
[12:20:35] prokka_EcoliK12/K12.txt
[12:20:35] prokka_EcoliK12/K12.log
[12:20:35] prokka_EcoliK12/K12.err
[12:20:35] prokka_EcoliK12/K12.tbl
[12:20:35] prokka_EcoliK12/K12.tsv
[12:20:35] Annotation finished successfully.
[12:20:35] Walltime used: 4.85 minutes
[12:20:35] If you use this result please cite the Prokka paper:
[12:20:35] Seemann T (2014) Prokka: rapid prokaryotic genome annotation. Bioinformatics. 30(14):2068-9.
[12:20:35] Type 'prokka --citation' for more details.
[12:20:35] Share and enjoy!
```

