## ANGSD
* Reminder to take notes, share info here: [Etherpad](https://pad.carpentries.org/2019-Oct-SMSC)

Now that we have the bam files from yesterday, we are going to run [ANGSD](http://www.popgen.dk/angsd/index.php/ANGSD). ANGSD can perform a variety of population genomics analyses. Today, we are going to use it to look at population structure and to estimate a site frequency spectrum (or allele frequency spectrum).

First, you need to add an additional `bam` file to your `variants` directory. This is a `bam` file of the raw reads for the reference, mapped back to its own genome. This will allow us to include it in our population structure analyses and demographic analyses. You can `cd` into your `pop_gen/variants` directory and copy it using:

`/scratch/genomics/frandsenp/SMSC/pop_gen/variants/ref_siskin.sorted.bam.mdup.bam* .`

The above command will copy both the bam file and its index file.

### 1. Generate genotype likelihoods
* We're going to make a PCA with ANGSD genotype calls.

* Create a new directory in your `pop_gen` directory called `angsd`.
* Change into `angsd`. We're going to create a file called `bamlist.txt`. This file will have relative paths to all of the bam files that we will use for this step. 
	+ Once in the `angsd` directory, you can create this list with the command:
`$ ls ../variants/*mdup.bam > bamlist.txt`

* When you create it, it should look something like the following:

	```
	../variants/JH-12872_AGTCAA.sam.sorted.bam.mdup.bam
	../variants/MB-12866_GTTTCG.sam.sorted.bam.mdup.bam
	../variants/MB-12867_TTAGGC.sam.sorted.bam.mdup.bam
	../variants/MB-12868_TGACCA.sam.sorted.bam.mdup.bam
	../variants/MB-S5_CAGATC.sam.sorted.bam.mdup.bam
	../variants/MB-S6_ACTTGA.sam.sorted.bam.mdup.bam
	../variants/MB-S7_GATCAG.sam.sorted.bam.mdup.bam
	../variants/MB-S8_TAGCTT.sam.sorted.bam.mdup.bam
	../variants/MB-S9_GGCTAC.sam.sorted.bam.mdup.bam
	```

* Now `cd` into your jobs directory and create a new job file. Call it `angsd_geno_like.job`.
	+ You should choose 4GB of RAM per CPU and 24 CPUs. The `ANGSD` command for to call the genotypes for a PCA is:
	+ **module**: `bioinformatics/angsd`
	+ **command**:

		```
		angsd -GL 2 -out ../angsd/PCA -nThreads $NSLOTS \
		-doGlf 2 -doMajorMinor 1 -SNP_pval 1e-6 -doMaf 1 \
		-bam ../angsd/bamlist.txt
		```

* There are a lot of options here. You can look at them more in depth in the ANGSD documentation linked above. For the log file, you should use something like `../logs/angsd_geno_like.log`.

* When it is complete, look in the `angsd` directory that you created earlier. You should have three new files: `PCA.arg`, `PCA.beagle.gz`, and `PCA.mafs.gz`. For the next step, we're going to generate a covariance matrix using the beagle file (which contains the genotype likelihoods) using `PCAngsd`.

### 2. Generate a covariance matrix using `PCAngsd`
* In this step, we're going to use the `PCA.beagle.gz` file to create a covariance matrix in `PCAngsd`.
	+ **module**: `bioinformatics/pcangsd`
	+ **command**: `pcangsd.py -beagle ../angsd/PCA.beagle.gz -o ../angsd/siskin_PCA -threads $NSLOTS` 
* After this is finished running, you should have a file in the `angsd` directory called `siskin_PCA.cov`. This is your covariance matrix and can be used to generate your PCA.

### 3. Create your PCA plot
* You can plot the PCA with an `R` script that can be found in `/scratch/genomics/frandsenp/SMSC/pop_gen/scripts/PCA_plot.r`. In order to run it, you need to have a list with the sample names in the same order as the original `bamlist`, `siskin_pop.txt`, in the same directory as the both the script and your covariance matrix. You can copy this file from `/scratch/genomics/frandsenp/SMSC/pop_gen/angsd/siskin_pop.txt`.
	+ **module**: `bioinformatics/R/3.6.0_conda`
	+ **command**: `Rscript PCA_plot.r`
* Now you can download your PCA plot and take a look at it.

### 4. Run Admixture analysis with `NGSAdmix`

* `NGSAdmix` is now part of the `ANGSD` package and can perform admixture analysis using the beagle file of genotype likelihoods that you generated in step 1. `cd` into your `jobs` directory and create a job file with the following commands. Alternatively, since it is fast on data from a single scaffold, a `qrsh` session would suffice. The parameter that you need to think about here is `K`, which as you learned in the lecture is the number of ancestral populations. Since we have two populations in this dataset, one from Venezuela and one from Guyana, we will set `K` to 2.
	+ **module**: `bioinformatics/angsd`
	+ **command**: `NGSadmix -likes ../angsd/PCA_beagle.gz -K 2 -o ../angsd/siskin_admix -P $NSLOTS`

* When the job is finished, you will have a couple of new files in the `angsd` directory. `siskin_admix.fopt.gz` is a gzipped file that contains the estimated allele frequencies for each population for each locus. `siskin_admix.qopt` contains the estimated admixture proportions for each individual.

### 5. Create simple admixture plot
* Now we will create a simple admixture plot using our results from `NGSadmix`
* Copy the script, `admix_plot.r` from `/scratch/genomics/frandsenp/SMSC/pop_gen/scripts/admix_plot.r` to your `angsd` directory.
* Make sure that `siskin_pop.txt` is still in your directory. This is another fast script, which is fine to run with `qrsh`.
	+ **module**: `bioinformatics/R/3.6.0_conda`
	+ **command**: `Rscript admix_plot.r`
* You will now have a plot in an image called `siskin_admix.png`. Feel free to download it and take a look. You will notice that, for these populations, there is very little evidence of admixture.
