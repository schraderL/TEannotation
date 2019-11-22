# De-novo repeat prediction with RepeatScout

## Setup environment
- add ```nseg``` to path (otherwise RepeatScout's ```filter-stage-1.prl``` will produce an empty output)

```bash
# add RepeatScout
export PATH=$PATH:/global/projects/programs/source/RepeatScout-1
export PATH=$PATH:"/global/projects/programs/source/censor-4.2.29/src/lcfilter/nseg/"
```
## Prepare folders and files
```bash
# create working directory
base=/global/homes/jg/schradel/data/TEannotation/RS
mkdir $base
cd $base

# Retrieve genome to process and clean up the fasta headers
cat <path_to_genome.fa> | cut -f 1 -d "|" > genome.fa
# define genome file for later use
genome=$(readlink -f genome.fa)

```

## Run repeatScout for denovo repeat annotation
see https://openwetware.org/wiki/Wikiomics:Repeat_finding#RepeatScout
```bash
# make sure nseg is in $PATH
which nseg
# build lmer_table
build_lmer_table -sequence $genome -freq $genome.lmer.frequency
# run RepeatScout
RepeatScout -sequence $genome -output $genome.repeats.fas  -freq $genome.lmer.frequency
# filter results
filter-stage-1.prl $genome.repeats.fas > $genome.repeats.fas.filtered_1
```

## Run PASTE-classifier (from REPET) to annotate repeats
### Prepare environment for PASTEC
```bash
# datafolder = folder with reference libraries (e.g. ProfilesBankForREPET_Pfam27.0_GypsyDB.hmm)
export datafolder="/global/homes/jg/schradel/data/REPET"

# add REPET_PATH to PATH
export REPET_PATH=/global/projects/programs/source/REPET_linux-x64-2.5
export PATH=$REPET_PATH/bin:$PATH
# ADD libraries to $LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/global/projects/programs/source/libs/
# add REPET to PYTHONPATH
export PYTHONPATH=$REPET_PATH:$PYTHONPATH

# Define MYSQL settings
export REPET_HOST=ebbsrv05
export REPET_USER=schradel
export REPET_PW=schr4d3l
export REPET_DB=REPET_schradel2
```

### retrieve database files for annotation
```
cd $base
ln -s $datafolder/ProfilesBankForREPET_Pfam27.0_GypsyDB.hmm .
ln -s $datafolder/RepBase20.05_REPET.embl/repbase20.05_ntSeq_cleaned_TE.fa .
ln -s $datafolder/RepBase20.05_REPET.embl/repbase20.05_aaSeq_cleaned_TE.fa .
```


# create PASTEClassifier.cfg
There are some things that need to be specified for each genome
- Define ```project_name```
- Define ```project_dir```

Also, there is an option ```remove_redundancy``` which could be useful.

Further, we could implement a scan for rDNA, host-genes, and tRNAs

```bash
[repet_env]
repet_version: 1.0
repet_host: ebbsrv05
repet_user: schradel
repet_pw: schr4d3l
repet_db: REPET_schradel2
repet_port: 3306
repet_job_manager: SGE

[project]
project_name: <set name or project>
project_dir: <set to $base>

[detect_features]
term_rep: yes
polyA: yes
tand_rep: yes
orf: yes
blast: blastplus
TE_BLRn: yes
TE_BLRtx: yes
TE_nucl_bank: repbase20.05_ntSeq_cleaned_TE.fa
TE_BLRx: yes
TE_prot_bank: repbase20.05_aaSeq_cleaned_TE.fa
HG_BLRn: no
HG_nucl_bank: <bank_of_host_genes>
TE_HMMER: no
TE_HMM_profiles: ProfilesBankForREPET_Pfam27.0_GypsyDB.hmm
TE_HMMER_evalue: 10
rDNA_BLRn: no
rDNA_bank: <bank_of_rDNA_sequences_from_eukaryota>
tRNA_scan: yes
TRFmaxPeriod: 15
clean: yes

[classif_consensus]
max_profiles_evalue: 1e-3
min_TE_profiles_coverage: 20
min_HG_profiles_coverage: 75
max_helitron_extremities_evalue: 1e-3
min_TE_bank_coverage: 5
min_HG_bank_coverage: 95
min_rDNA_bank_coverage: 95
min_HG_bank_identity: 90
min_rDNA_bank_identity: 90
min_SSR_coverage: 0.75
max_SSR_size: 100
remove_redundancy: yes
min_redundancy_identity: 95
min_redundancy_coverage: 98
rev_complement: no
add_wicker_code: yes
add_noCat_bestHitClassif: yes
clean: yes
```

## Run ```PASTEclassifier.py```
```bash
# repeatscout fasta cleanup
cut -f 1 -d " " $genome.repeats.fas.filtered_1|sed 's/=/./g'  > $genome.RSlib.fa
PASTEClassifier.py -i $genome.RSlib.fa -C PASTEClassifier.cfg
```
