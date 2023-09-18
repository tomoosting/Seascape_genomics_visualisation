# EMC_job_application
Source code Tom Oosting

Contents
1. Sbatch array script for genotyping by chromosome using BCFtools  
2. Rmarkdown file for genome scan
3. Rmarkdown file visualisation of genotype-by-environment analyses (SeaScape Genomics)

### 1. SBATCH array
The following script uses bam files and a file with sample and populaiton information to generare an AllSites VCF.
Resulting VCF file is tab indexed an analysed using pixy.
Output from pixy is used in R to scan the genome for signs of selection.
```
#!/bin/bash
#SBATCH -a 1-24
#SBATCH --cpus-per-task=8
#SBATCH --mem-per-cpu=10G
#SBATCH --partition=parallel
#SBATCH --time=5-0:00
#SBATCH --job-name=pixy
#SBATCH -o /nfs/scratch/oostinto/stdout/pixy_%A_%a.out
#SBATCH -e /nfs/scratch/oostinto/stdout/pixy_%A_%a.err
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=tom.oosting@vuw.ac.nz

#activate conda environment for pixy
source activate pixy  

#load modules
module load htslib/1.9
module load bcftools/1.10.1

###run input
PROJECT=$1
SET=$PROJECT'_'$2
LG=${SLURM_ARRAY_TASK_ID}

###
POP=$SCRATCH/projects/$PROJECT/resources/sample_info/$SET'_pixy.tsv'
BAM=$SCRATCH/projects/$PROJECT/resources/bam_lists/$SET'_bam.list'
REF=$SCRATCH/projects/$PROJECT/resources/reference_genomes/nuclear/Chrysophrys_auratus.v.1.0.all.assembly.units.fasta
REG=$( head -n $LG $REF.fai | tail -n 1 | cut -f 1 )

###set paths
VCF=$SCRATCH/projects/$PROJECT/data/snp/$SET/$SET
OUT=$SCRATCH/projects/$PROJECT/output/$SET/pixy
TMP=$SCRATCH/projects/$PROJECT/output/$SET/pixy/tmp

mkdir -p $TMP

#genotype and filter
echo "performing genotyping and filtering"
bcftools mpileup 	-f $REF 							\
					-b $BAM 							\
					-r $REG								\
					-a 'INFO/AD,FORMAT/AD,FORMAT/DP'	|
bcftools call 		-m 									\
					-Ou 								\
					-f GQ 								|	 
bcftools +fill-tags	-Ou 								\
					-- -t AC,AF,AN,MAF,NS				|
bcftools filter		-Ou									\
					-S .								\
					--exclude 'FMT/DP<3 | FMT/GQ<20'	|
bcftools view		-Oz									\
					-M2									\
					--exclude 'STRLEN(REF)!=1 | STRLEN(ALT) >=2 | QUAL<600 | AVG(FMT/DP)<8 | AVG(FMT/DP)>25 ' 	\
					-o $TMP/$REG'_'$SET'_raw.allsites.vcf.gz'

#create index
echo "creating index"
tabix -f $TMP/$REG'_'$SET'_raw.allsites.vcf.gz'

# run pixy
echo "running pixy"
pixy	--stats pi dxy fst								\
		--vcf $TMP/$REG'_'$SET'_raw.allsites.vcf.gz'	\
		--populations $POP								\
		--window_size 5000								\
		--n_cores 4										\
		--output_folder $OUT							\
		--output_prefix $REG'_'$SET	
```
### 2. genome scan (genome_scan.Rmd) 
This Rmarkdown script utilises a different package to perform a genome scan (i.e. PopGenome).
The script first imports the associated functions.R containing custom fuctions I've created.
![alt text](./Figures/snapper_norm_qc_slw5000_genome_scan.png)

### 3. Visualisation of SeaScape analyses
Figure prodiced by seascape analyses
![alt text](./Figures/snapper_382_qc_thin5000_heterogeneous_pH_PC1_joined.png)
