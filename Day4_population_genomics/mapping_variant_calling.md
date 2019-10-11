### 1. Running Bowtie2
* Reminder to take notes, share info here: [Etherpad](https://pad.carpentries.org/2019-Oct-SMSC)
* Create a pop_gen folder inside your ```./SMSC``` directory, then create subdirectories: jobs, logs, genome, variants
* For this part of the tutorial, we're each going to work with a single long scaffold (xx.pilon.fasta) from the Red Siskin assembly. 
	+ It is here: ```/data/genomics/workshops/SMSC_2019/Contig3141_pilon.fasta```
	+ Copy this scaffold to your new genome folder in your ```/pool/genomics/``` space. Eg. ```cp /data/genomics/workshops/SMSC_2019/Contig3141_pilon.fasta ./genome``` if you are inside the pop_gen folder.
* First we'll need to index the fasta file (reference scaffold) before we can map raw data to it. Create a job file for this:
	+ **module**: ```bioinformatics/bowtie2```
	+ **command**: ``bowtie2-build --threads $NSLOTS <SCAFFOLD.fasta> NAME``
	+ Hint: it doesn't need much RAM or time.
	+ It will produce files that end in bt2.
* Then we can start mapping the reads to our scaffolds.
	+ There are 9 individuals plus the one from which we are assembling the genome.
	+ The read data are here (each individual has a pair of fq.gz): ```/scratch/genomics/dikowr/siskin_raw_data/resequencing```
	+ Don't copy it to your space.
	+ Create job files to map each individual's reads to your scaffold.
	+ **module**: ```bioinformatics/bowtie2```
	+ Here is an example **command** (this will need editing to make it work for your data): ```bowtie2 --local --very-sensitive-local -N 1  -I 100 -X 800 -x ../genome/siskin \
	  -p $NSLOTS --phred33 --rg-id "$1" --rg SM:"$1" --rg PL:"ILLUMINA" --rg LB:"hiseq.phred33" \
	  -1 /scratch/genomics/dikowr/siskin_raw_data/resequencing/$1_R1_all.fq.gz -2 /scratch/genomics/dikowr/siskin_raw_data/resequencing/$1_R2_all.fq.gz \
	  -S ../variants/$1.sam 2> ../logs/$1.stat```
	+ Run the shell script with ```sh 2b_bowtie_map.sh```
	+ Check out the output file using ```head```

### 2. Manipulating the Bowtie2 output 
* In order to get the Bowtie2 outputs ready to go for variant calling, we'll have to do some file manipulation.
* First, we'll have to convert our sam file output to bam format.  
* Create a job file to do this.
	+ **module**: ```bioinformatics/samtools```
	+ **command**: ```samtools view -b <YOUR_OUTPUT.sam> > <YOUR_OUTPUT.bam>```  
	+ Try ```head``` on the output bam file to see what happens.
* Next we'll need to sort the bam file.
	+ **module**: ```bioinformatics/samtools```
	+ **command**: ```samtools sort <YOUR_BAM.bam> -o <YOUR_BAM_sorted.bam>```  
* Or combine the two into one command
  * **command**: ```samtools view -u $1 | samtools sort -O bam -o $1.sorted.bam```
  * **to run the job file**: ```for i in ../variants/*sam; do qsub 3_sortedbam.job $i; done```

### 3. Mark Duplicates with picard-tools (not necessary if library is PCR-free)
* For future steps, we will need an sequence dictionary and fasta index.
* Create a job file to create a fasta index on your scaffold:
	+ **module**: ```module load bioinformatics/samtools```
	+ **command**: ```samtools faidx <YOUR_SCAFFOLD.fasta>```
* Create a job file to create a sequence dictionary:
	+ **module**: ```module load bioinformatics/picard-tools```
	+ **command**: ```runpicard CreateSequenceDictionary R=../genome/<YOUR_SCAFFOLD.fasta> O=../genome/<YOUR_SCAFFOLD.dict>```
* Now we can create a job file to mark duplicates:
	+ **module**: ```module load bioinformatics/picard-tools/2.20.6```
	+ **command**: ```runpicard MarkDuplicates I=$NAME M=$NAME.metric.txt O=$NAME.mdup.bam \
	  MAX_FILE_HANDLES_FOR_READ_ENDS_MAP=1000 SORTING_COLLECTION_SIZE_RATIO=0.25```
	+ To run the job file: ```for i in ../variants/*sorted.bam; do qsub 5_picard_mark_dup.job $i; done```

### 4. Realign indels before calling variants (not necessary after GATK 3.7 if using HaplotypeCaller)
* Now we'll look for regions with indels in our bam files and realign them.
* Index the marked bam files:
	+ **module**: ```module load bioinformatics/samtools```
	+ **command**: ```samtools index <YOUR_SORTED_BAM_marked.bam>```
	+ This will produce files that end with bai.
	+ To run the part on indexing bam files: ``` for i in ../variants/*mdup.bam; do qsub 6_samtools_index.job $i; done```
* Create a job file to use GATK RealignerTargetCreator:
	* Create a folder called ./realign_target
	* **module**: ```module load bioinformatics/gatk```
	* **command**: ```rungatk -T RealignerTargetCreator -R <YOUR_SCAFFOLD.fasta> -I <YOUR_SORTED_BAM_marked.bam> -o <YOUR_SORTED_BAM_marked.bam>.list```
* Create a job file to use GATK IndelRealigner:
	+ **module**: ```module load bioinformatics/gatk```
	+ **command**: ```rungatk -T IndelRealigner -R <YOUR_SCAFFOLD.fasta> -I <YOUR_SORTED_BAM_marked.bam> -o indels-<YOUR_SORTED_BAM_marked.bam> -targetIntervals <YOUR_SORTED_BAM_marked.bam>.list```
* Index the indel-marked bam files:
	+ **module**: ```module load bioinformatics/samtools```
	+ **command**: ```samtools index <indels-YOUR_SORTED_BAM_marked.bam>```

### 5. Call variants with GATK
* First, we will run GATK HaplotypeCaller on each bam file individually:
	+ **module**: ```module load bioinformatics/gatk```
	+ **command**: ```rungatk -T HaplotypeCaller -ERC GVCF -pcrModel NONE -R <YOUR_SCAFFOLD.fasta> -I <YOUR_SORTED_BAM_marked.bam> -o <YOUR_BAM.g.vcf> -nct $NSLOTS```
	+ This job might need to be run on himem.
	+ To run: ```for i in ../variants/*mdup.bam; do qsub 9_gatk_hapcall.job $i; done```
* Now we'll run GATK GenotypeGVCFs across all vcf files:
	+ **module**: ```module load bioinformatics/gatk```
	+ **command**: ```rungatk -T GenotypeGVCFs -R <YOUR_SCAFFOLD.fasta> -V HERE_LIST_ALL_YOUR_VCF_FILES -o VCF```
	+ This job might need to be run on himem.
* Use ```head``` to look at the VCF file.

### 6. Select and filter variants with GATK

* Use the job file 11_gatk_select_filter_Gfilter_siskin_only.job to select only SNPs and conduct variant- and genotype-level filtering:
	+ **module**: ```module load bioinformatics/vcftools```
You can also use vcftools inside qrsh to carry out some of the same functions.
	+ **module**: ```module load bioinformatics/vcftools```
To remove indels and keep only SNPs
	+ **command**: ```vcftools --vcf siskin_raw.vcf --remove-indels --recode --recode-INFO-all --out siskin_raw_SNP```
	
	Removes all sites with a FILTER flag other than PASS
	
	+ **command**: ```vcftools --vcf siskin_SNP_filter0_DP8GQ13.vcf --recode --recode-INFO-all --remove-filtered-all --out siskin_SNP_filter0_DP8GQ13_PassOnly```
	
	Removes all sites with a FILTER flag other than PASS and only SNPs that are not missing in all individuals
	
	+ **command**: ```vcftools --vcf siskin_SNP_filter0_DP8GQ13.vcf --recode --recode-INFO-all --remove-filtered-all --out siskin_SNP_filter0_DP8GQ13_PassOnlyPOP_nomiss --max-missing 1```
	
	Find missingness across individuals
	
	+ **command**: ```vcftools --vcf  siskin_SNP_filter0_DP8GQ13_PassOnly --missing-indv``` 

### 7. View your vcf file with IGV
* Transfer a filtered vcf, scaffold and its index (fai) to your local computer
* Go to https://igv.org/ and open IGV Web App
* Click on genome and load your scaffold and its fai file
* Click on Track and load you vcf file