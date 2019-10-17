## _De novo_ transcriptome assembly with Trinity

Make a new subdirectory in your ```/scratch/genomics/<username>/smsc_2019/rnaseq``` directory for running trinity.

```
$ mkdir trinity
$ cd trinity
```

####Trinity _de novo_ assembly

Now we will start the Trinity run.

First open the [QSubGen web application](https://hydra-admin01.si.edu/tools/QSubGen/).

- Choose medium time limit and reserve 6GB of memory.
- Select 'multi-thread' and choose 10 CPUs.
- Total memory requested will be 60GB
- In the modules field, type ```trinity``` and the select the module ```bioinformatics/trinity/2.8.5```.
- In job specific commands, type:
          
      	Trinity --seqType fq \
      	--left ../data/RNA_Eye_1.fastq \
      	--right ../data/RNA_Eye_2.fastq \
      	--max_memory 60G --CPU $NSLOTS \
      	--output RNA_Eye.trinity \
      	--trimmomatic --full_cleanup 

- Choose a descriptive job name then click on 
- ```Check if OK```
- If it passes, either save it and upload it to Hydra, or copy the text and paste it directly into your favority text editor.

Now save your text file into your ```/scratch/genomics/<username>/smsc_2019/rnaseq/trinity/``` directory as ```trinity.job```.

Now submit your job with the command: ```qsub trinity.job```

Soon your transcriptome assembly will be finished!

```
Results of all tutorials can be found here:
/data/genomics/workshops/smsc/RNA_Seq/SMSC_results.tar.gz
```