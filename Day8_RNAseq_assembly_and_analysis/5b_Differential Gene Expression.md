## Differential Gene Expression with edgeR

_Note: First we need to load our modules_

Once you are logged in to hydra:

Make sure that you are in your ```/scratch/genomics/<username>/smsc_2019/rnaseq``` directory.

Create a working directory and enter that directory:

```
$ mkdir dge
$ cd dge/
```

####Create sample text file

You will need to create a tab delimited text file containing information for your different samples. In the first column, you will have the name of the condition and, in the second column, you will enter the name of the sample. Start a new text file, samples.txt, with nano.

```
$ nano samples.txt
```

Enter the following (keep in mind that the condition and names should be separated by tabs!).

```
GSNO	GSNO_SRR1582648.RSEM
GSNO	GSNO_SRR1582647.RSEM
GSNO	GSNO_SRR1582646.RSEM
WT	wt_SRR1582651.RSEM
WT	wt_SRR1582649.RSEM
WT	wt_SRR1582650.RSEM
```

Since software can be very picky about whether you specified config files correctly, it is sometimes good to check that you did, indeed, enter the correct characters. You can view special characters with:

```
$ cat -te samples.txt
```

If your file was specified correctly, it should look like this:

```
GSNO^IGSNO_SRR1582648.RSEM$
GSNO^IGSNO_SRR1582647.RSEM$
GSNO^IGSNO_SRR1582646.RSEM$
WT^Iwt_SRR1582651.RSEM$
WT^Iwt_SRR1582649.RSEM$
WT^Iwt_SRR1582650.RSEM$
```

```^I``` characters are tabs and ```$``` characters are newlines. Make sure that your text file looks like the example above when using ```cat -te```. If it doesn't, you'll need to edit it until it does.

####Detect differentially expressed transcripts in ```edgeR```

Now we are going to use the ```run_DE_analysis.pl``` script that is included with the ```Trinity``` package to detect differentially expressed transcripts in edgeR. 

Create a new job file, and select the short queue and 2GB of RAM. Load the Trinity module (bioinformatics/trinity/2.8.5) and the R module (bioinformatics/R). The command will look like this:

```
run_DE_analysis.pl \
      --matrix ../quant/Trinity_trans.isoform.TMM.EXPR.matrix \
      --samples_file samples.txt \
      --method edgeR \
      --output edgeR_trans
```

Save the job file to your ```/pool/genomics/<username>/RNAseq_SMSC``` directory and submit it to the cluster. Once it is finished, there will be a new directory called ```edgeR_trans```. Take a look at its contents:

```
$ ls -lh edgeR_trans
```

There should be four files in the directory:

```
-rw-rw-r-- 1 gonzalezv nmnh_ggi  30K Oct 16 22:18 Trinity_trans.isoform.TMM.EXPR.matrix.GSNO_vs_WT.edgeR.count_matrix
-rw-rw-r-- 1 gonzalezv nmnh_ggi  66K Oct 16 22:18 Trinity_trans.isoform.TMM.EXPR.matrix.GSNO_vs_WT.edgeR.DE_results
-rw-rw-r-- 1 gonzalezv nmnh_ggi  15K Oct 16 22:18 Trinity_trans.isoform.TMM.EXPR.matrix.GSNO_vs_WT.edgeR.DE_results.MA_n_Volcano.pdf
-rw-rw-r-- 1 gonzalezv nmnh_ggi 1.5K Oct 16 22:18 Trinity_trans.isoform.TMM.EXPR.matrix.GSNO_vs_WT.GSNO.vs.WT.EdgeR.Rscript
```

The file ```Trinity_trans.isoform.counts.matrix.GSNO_vs_WT.edgeR.DE_results``` contains the results from comparing the GSNO condition to the wt condition. Take a look:

```
$ head edgeR_trans/Trinity_trans.isoform.TMM.EXPR.matrix.GSNO_vs_WT.edgeR.DE_results | column -t
```


```
sampleA                 sampleB  logFC  logCPM             PValue            FDR
TRINITY_DN516_c0_g1_i1  GSNO     WT     15.556185663025    11.5751958145976  1.57981412556518e-15  1.0252993674918e-12
TRINITY_DN283_c0_g1_i1  GSNO     WT     -15.0635550284712  11.0829416025116  1.53687398018169e-12  3.58833982749458e-10
TRINITY_DN587_c0_g1_i1  GSNO     WT     -5.8639475757608   12.565360171226   1.65870870300212e-12  3.58833982749458e-10
TRINITY_DN425_c0_g1_i1  GSNO     WT     -14.8841645717427  10.9037238108485  3.70480440604664e-11  6.01104514881067e-09
TRINITY_DN142_c1_g1_i1  GSNO     WT     -14.7485988347704  10.7683035572247  2.25283170089928e-10  2.92417554776727e-08
TRINITY_DN271_c0_g1_i1  GSNO     WT     -14.8697864252781  10.8893604551253  3.19791159134044e-10  3.1222329737815e-08
TRINITY_DN278_c0_g1_i1  GSNO     WT     -14.7453092944008  10.7650177115708  3.36758564198313e-10  3.1222329737815e-08
TRINITY_DN416_c0_g1_i1  GSNO     WT     14.8516139872303   10.8712084777169  5.94890771964078e-10  4.82605138755859e-08
TRINITY_DN486_c0_g1_i1  GSNO     WT     -14.6607394507756  10.6805460110796  6.9051644858468e-10   4.97939083479397e-08
```

As you can see, ```edgeR``` calculates log fold change (```logFC```), the log counts per million (```logCPM```), the p-value from the exact test (```PValue```), and the false discovery rate (```FDR```). 

_Note: Since there is no header for gene name, the headers are shifted one column to the right, i.e. ```logFC``` should be over the first column of floating point numbers._

```edgeR``` also generated MA and Volcano plots for these data. We will now download them to our computer. If you are using Mac or Linux, we will do this with the ```scp``` command. Open a new terminal window and ```cd``` to the directory that you wish to download the files to. On Mac, I often download to my ```Downloads``` directory. You can go there with:

```
$ cd ~/Downloads
```

Now download the plot:

```
$ scp <username>@hydra-login01.si.edu:/scratch/genomics/<username>/smsc_2019/rnaseq_SMSC/edgeR_trans/*.pdf .
```

Go ahead and open it to examine its contents.

![Volcano Plot](Trinity_trans.isoform.TMM.EXPR.matrix.GSNO_vs_WT.edgeR.DE_results.MA_n_Volcano.pdf)

The points that are in red are determined to be significant with an ```FDR``` <= 0.05. To read more about these tests, you can follow the citations on the [edgeR bioconductor page](https://bioconductor.org/packages/release/bioc/html/edgeR.html).

You might wonder what you can do with these data. Luckily, Trinity also includes scripts to extract differentially expressed transcripts and to create heatmaps.

Change directories into your ```edgeR_trans``` directory:

```
$ cd edgeR_trans
```

Now we will extract any transcript that is 4-fold differentially expressed between the two conditions at a significance of ```<= 0.001```.

Make another job file and choose the short queue and reserve the default RAM (1GB). Load the ```bioinformatics/trinity/2.8.5``` module and the ```bioinformatics/R``` module. Your command will be:

```
analyze_diff_expr.pl \
      --matrix ../../quant/Trinity_trans.isoform.TMM.EXPR.matrix \
      --samples ../samples.txt \
      -P 1e-3 -C 2 
```

This command will filter transcripts based on pvalue of less than Several files will be written as a part of this job. One is called ```diffExpr.P1e-3_C2.matrix```. You can count the number of differentially expressed genes at this threshold by counting the number of lines:

```
$ wc -l diffExpr.P1e-3_C2.matrix
```

You should subtract 1 from the number since there is a header line.

This script also generates a heatmap that compares the differentially expressed transcripts. The file is called, ```diffExpr.P1e-3_C2.matrix.log2.centered.genes_vs_samples_heatmap.pdf```.

Download that file and examine it on your computer.

_Hint: you can use ```scp``` as above. Or you can use a GUI interface like Filezilla/Cyberduck._

Examine the Sample Correlation: 

![Differential Expression Sample Correlation](diffExpr.P1e-3_C2.matrix.log2.centered.sample_cor_matrix.pdf)

Now examine the heatmap of DGE: 

![Differential Expression Heatmap](diffExpr.P1e-3_C2.matrix.log2.centered.genes_vs_samples_heatmap.pdf)

You can use the heatmap to compare the two conditions. The left columns with the turquoise line on top are those under wt and the right columns under the red line are under GSNO. Upregulated expression is in yellow and downregulated expression is in purple. This is a nice visual way to compare expression across conditions.

####View transcript clusters

You can also cut the dendrogram to view transcript clusters that share similar expression profiles. To do this, run the following command into a job file. Be sure to load the ```bioinformatics/trinity/2.6.6``` module and choose a serial job with 1GB of RAM:

```
define_clusters_by_cutting_tree.pl --Ptree 60 -R diffExpr.P1e-3_C2.matrix.RData
```

You should have a new output that looks like the following graph, which shows transcripts with similar expression profiles:

![My cluster plots](my_cluster_plots.pdf)

####Now run on genes

Now we will run differential expression analysis on the gene level. This will be very similar to the isoform analysis, but we will use the follow command:

```
run_DE_analysis.pl \
      --matrix T../../quant/Trinity_trans.gene.TMM.EXPR.matrix\
      --samples_file samples.txt \
      --method edgeR \
      --output edgeR_gene
```

You can also run the other downstream analyses, but you should replace ```Trinity_trans``` with ```Trinity_genes``` in the commands.