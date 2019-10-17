## Transcript quantification

We will be using RSEM to quantify the expression levels of the transcripts that have been assembled by Trinity. But because we do not have proper biological replicates for the Red Sisken RNAseq data, we will be using _Candida_ _glabrata_ transcriptome data. The paper from which these data are derived examined *C. glabrata* in two conditions, nutrient rich (wt) and under nitrosative stress (GNSO).

Make sure that you are in your ```/scratch/genomics/<username>/smsc_2019/rnaseq``` directory.

Create a subdirectory for these analyses called "quant."

```
$ mkdir quant
$ cd quant 
```

To use RSEM, we will use the wrapper script included in the Trinity package called ```align_and_estimate_abundance.pl```.

This script makes it very easy to take Trinity output and run RSEM.

Create another job file with [QSubGen](https://hydra-admin01.si.edu/tools/QSubGen/). This will be a serial job and you should reserve 4GB of memory. It will also run fine in the short queue.

Load the trinity module (bioinformatics/trinity/2.8.5), the R module (bioinformatics/R/3.6.1), and enter the following program command:

```
align_and_estimate_abundance.pl --seqType fq \
--left ../data/diff_ex/GSNO_SRR1582648_1.fastq \
--right ../data/diff_ex/GSNO_SRR1582648_2.fastq  \
--transcripts ../data/diff_ex/trinity_out_dir.Trinity.fasta \
--est_method RSEM  --aln_method bowtie \
--trinity_mode --prep_reference --coordsort_bam \
--output_dir GSNO_SRR1582648.RSEM

```

Save the script into a file called ```trinity_rsem_GNSO_SRR1582648.job```. Since there are six biological replicates, we'll need to make six of these files -- one for each replicate.

We can take some time to create the five other files for the other treatments or ***we can execute this job as a loop (see advanced below).*** Be sure to change both the names of the reads, and the ```--output_dir``` according to the sample name. Double check that you are calling both the correct reads and using the correct names. It is always important to check twice, run once.

Now you can submit each of these jobs using ```qsub job_file_name```. This is the magic of parallel computing clusters--you can run many jobs at once.

***Advanced: Make a loop script that runs this for all the samples.***

<details><summary>SOLUTION</summary>
<p>

Make a list of samples as a text file (list.txt) using `nano`: 

```
GSNO_SRR1582646
GSNO_SRR1582647
GSNO_SRR1582648
wt_SRR1582649
wt_SRR1582650
wt_SRR1582651
```

Example loop: 

```
for x in `cat list.txt` 

do 

align_and_estimate_abundance.pl --seqType fq \
--left ../data/diff_ex/${x}_1.fastq \
--right ../data/diff_ex/${x}_2.fastq  \
--transcripts ../data/diff_ex/trinity_out_dir.Trinity.fasta \
--est_method RSEM  --aln_method bowtie \
--trinity_mode --prep_reference --coordsort_bam \
--output_dir ${x}.RSEM

done
```
</p>
</details>

These jobs should run pretty fast. Once they are finished, you can check the output with, e.g.:

```$ head GSNO_SRR1582648.RSEM/RSEM.genes.results | column -t```

Your output should look something like:

```
gene_id              transcript_id(s)        length  effective_length  expected_count  TPM      FPKM
TRINITY_DN100_c0_g1  TRINITY_DN100_c0_g1_i1  253.00  123.83            2.00            1014.01  4866.18
TRINITY_DN103_c0_g1  TRINITY_DN103_c0_g1_i1  524.00  394.48            57.00           9071.81  43534.98
TRINITY_DN103_c1_g1  TRINITY_DN103_c1_g1_i1  152.00  27.95             3.00            6739.18  32340.83
TRINITY_DN104_c0_g1  TRINITY_DN104_c0_g1_i1  174.00  47.05             0.00            0.00     0.00
TRINITY_DN105_c0_g1  TRINITY_DN105_c0_g1_i1  221.00  92.16             0.00            0.00     0.00
TRINITY_DN105_c1_g1  TRINITY_DN105_c1_g1_i1  238.00  108.94            1.00            576.30   2765.60
TRINITY_DN107_c0_g1  TRINITY_DN107_c0_g1_i1  161.00  35.48             1.00            1769.44  8491.42
TRINITY_DN108_c0_g1  TRINITY_DN108_c0_g1_i1  190.00  62.01             0.00            0.00     0.00
TRINITY_DN108_c1_g1  TRINITY_DN108_c1_g1_i1  195.00  66.79             1.00            940.01   4511.05
```

####Generate a transcript counts matrix and perform cross-sample normalization

Each rsem file holds the expression estimates for each of the samples. Now we will use these samples to create a counts matrix and to perform cross-sample normalization. The script included in the Trinity packages does this for you according to the TMM method. If you want to read more about this normalization method, you can do so in this paper, "[A scaling normalization method for differential expression analysis of RNA-seq data](http://genomebiology.biomedcentral.com/articles/10.1186/gb-2010-11-3-r25)."

Go ahead and generate a job file using the QSubGen. Assuming that you set up the remaining 5 jobs the same way that you did the first job, your job command would be:

```
abundance_estimates_to_matrix.pl --est_method RSEM \
	  --gene_trans_map ../data/diff_ex/trinity_out_dir.Trinity.fasta.gene_trans_map \
      --out_prefix Trinity_trans --name_sample_by_basedir \
      GSNO_SRR1582648.RSEM/RSEM.isoforms.results \
      GSNO_SRR1582646.RSEM/RSEM.isoforms.results \
      GSNO_SRR1582647.RSEM/RSEM.isoforms.results \
      wt_SRR1582649.RSEM/RSEM.isoforms.results \
      wt_SRR1582651.RSEM/RSEM.isoforms.results \
      wt_SRR1582650.RSEM/RSEM.isoforms.results
```

Choose a serial key and 2GB of memory. Either save the job file and transfer it to Hydra, or copy and paste the text into your terminal window with ```nano```. Subit the job.

This command will run estimates for both genes and isoforms. The output of this command will produce:

- RSEM.isoform.counts.matrix  : the estimated RNA-Seq fragment counts (raw counts)
- RSEM.isoform.TPM.not\_cross\_norm  : a matrix of TPM expression values (not cross-sample normalized)
- RSEM.isoform.TMM.EXPR.matrix : a matrix of TMM-normalized expression values
   
First lets look at the at the first 20 lines of the isoform counts. 
```
$ head -20 Trinity_trans.isoform.counts.matrix | column -t
```

The resulting output will look something like this:

```
GSNO_SRR1582648.RSEM    GSNO_SRR1582646.RSEM  GSNO_SRR1582647.RSEM  wt_SRR1582649.RSEM  wt_SRR1582651.RSEM  wt_SRR1582650.RSEM
TRINITY_DN103_c1_g1_i1  3.00                  0.00                  0.00                0.00                0.00                0.00
TRINITY_DN564_c0_g1_i1  6.00                  2.00                  3.00                2.00                5.00                2.00
TRINITY_DN96_c0_g1_i1   0.00                  2.00                  1.00                0.00                1.00                1.00
TRINITY_DN311_c0_g1_i1  5.00                  1.00                  4.00                0.00                0.00                0.00
TRINITY_DN374_c0_g1_i1  0.00                  0.00                  0.00                1.00                1.00                1.00
TRINITY_DN284_c0_g1_i1  0.00                  0.00                  4.00                9.00                9.00                15.00
TRINITY_DN321_c0_g1_i1  1.00                  5.00                  3.00                2.00                1.00                5.00
TRINITY_DN133_c0_g1_i1  2.00                  0.00                  0.00                2.00                0.00                0.00
TRINITY_DN21_c0_g1_i1   0.00                  1.00                  0.00                2.00                1.00                3.00
TRINITY_DN23_c0_g1_i1   0.00                  0.00                  0.00                5.00                3.00                8.00
TRINITY_DN173_c0_g1_i1  1.00                  1.00                  2.00                3.00                0.00                1.00
TRINITY_DN258_c2_g1_i1  0.00                  1.00                  1.00                9.00                3.00                5.00
TRINITY_DN526_c0_g1_i1  2.00                  5.00                  4.00                0.00                1.00                1.00
TRINITY_DN44_c1_g1_i1   0.00                  0.00                  0.00                4.00                1.00                4.00
TRINITY_DN632_c0_g1_i1  7.00                  1.00                  2.00                5.00                3.00                3.00
TRINITY_DN296_c0_g1_i1  14.00                 18.00                 13.00               37.00               28.00               36.00
TRINITY_DN220_c0_g1_i1  56.00                 63.00                 57.00               19.00               12.00               17.00
TRINITY_DN378_c0_g1_i1  1.00                  0.00                  0.00                4.00                1.00                1.00
TRINITY_DN176_c0_g1_i1  0.00                  0.00                  0.00                3.00                1.00                2.00

```

Now look at the output generated for the isoforms from the TMM normalized counts:

```
$ head -20 Trinity_trans.isoform.TMM.EXPR.matrix | column -t
```

The output from this file will look a bit different. As described in the paper linked to above, this normalization method assumes that most transcripts are not differentially expressed and linearly scales the values with that assumption in mind.

```
GSNO_SRR1582648.RSEM    GSNO_SRR1582646.RSEM  GSNO_SRR1582647.RSEM  wt_SRR1582649.RSEM  wt_SRR1582651.RSEM  wt_SRR1582650.RSEM
TRINITY_DN270_c0_g1_i1  0.000                 0.000                 0.000               1365.360            0.000               1383.917
TRINITY_DN275_c2_g1_i1  0.000                 1630.748              2267.950            801.320             0.000               0.000
TRINITY_DN84_c0_g1_i1   385.169               0.000                 666.522             733.131             841.754             387.554
TRINITY_DN638_c0_g1_i1  0.000                 1142.342              0.000               2181.399            3073.698            4466.226
TRINITY_DN187_c1_g1_i1  993.610               1924.992              1775.241            1865.203            1274.401            961.209
TRINITY_DN252_c0_g1_i1  1657.153              0.000                 0.000               523.324             639.099             3845.269
TRINITY_DN600_c0_g1_i1  0.000                 0.000                 0.000               2141.474            1503.270            1097.020
TRINITY_DN462_c0_g1_i1  1460.146              2923.206              1327.875            0.000               0.000               0.000
TRINITY_DN248_c0_g1_i1  267.652               0.000                 0.000               1277.388            836.795             1627.422
TRINITY_DN309_c0_g1_i1  0.000                 841.341               0.000               1649.333            0.000               1707.973
TRINITY_DN146_c0_g1_i1  0.000                 0.000                 0.000               0.000               3143.351            1136.775
TRINITY_DN217_c0_g1_i1  1318.177              1497.220              568.539             314.103             0.000               332.738
TRINITY_DN478_c0_g1_i1  773.547               0.000                 0.000               2186.149            949.179             3032.159
TRINITY_DN278_c0_g1_i1  0.000                 0.000                 0.000               2926.352            4372.631            2956.880
TRINITY_DN225_c0_g1_i1  2532.929              1778.327              1393.598            4326.008            3810.915            5712.654
TRINITY_DN391_c0_g1_i1  839.814               0.000                 0.000               790.105             3136.298            819.487
TRINITY_DN147_c0_g1_i1  851.913               0.000                 755.980             0.000               1063.222            830.682
TRINITY_DN562_c0_g1_i1  3266.998              2757.252              4194.239            2777.435            2185.174            3142.635
TRINITY_DN241_c0_g1_i1  459.494               1693.648              399.173             0.000               1547.556            459.963

```

We can also examine the generated matrices for genes. 

```
$ head -20 Trinity_trans.gene.counts.matrix | column -t
```
```
$ head -20 Trinity_trans.gene.TMM.EXPR.matrix | column -t
```


Now that we have expression values we can estimate some new statistics. We will use the expression quantification values to calculate the ExN50, which is restricted to only the most highly expressed transcripts. This is more informative of the quality of the assembly since it only uses the transcripts that have suitable coverage.

The file that you generate from QSubGen should choose the short queue and the default RAM. Load the ```bioinformatics/trinity/2.8.5``` module. The command that you should use is:

```
contig_ExN50_statistic.pl ../data/diff_ex/Trinity_trans.TMM.EXPR.matrix \
../data/diff_ex/trinity_out_dir.Trinity.fasta > ExN50.stats
```

When your job is completed, you will see a new output file called, ```ExN50.stats```. Go ahead and take a look at your new file with:

```
$ cat ExN50.stats | column -t
```

Your output should look like this:

```
Ex   ExN50  num_transcripts
2    470    1
4    470    2
6    329    3
7    470    4
9    453    5
11   453    6
13   470    7
14   453    8
16   453    9
17   416    10
19   453    11
20   416    12
22   416    13
23   416    14
24   416    15
26   416    16
27   453    17
28   459    18
30   470    19
31   470    20
32   470    21
33   459    22
34   459    23
35   470    24
36   459    25
37   470    26
38   470    27
39   470    28
40   470    29
41   470    30
42   470    31
43   470    32
44   470    34
45   470    35
46   470    36
47   473    38
48   470    39
49   473    41
50   485    42
51   489    44
52   489    46
53   489    47
54   489    49
55   489    51
56   489    53
57   489    55
58   489    57
59   489    59
60   489    62
61   489    64
62   489    66
63   489    69
64   485    72
65   473    74
66   470    77
67   470    80
68   470    84
69   468    87
70   468    90
71   468    94
72   459    97
73   456    101
74   453    105
75   443    109
76   443    113
77   439    117
78   427    122
79   422    126
80   420    131
81   420    136
82   416    142
83   416    148
84   416    154
85   415    160
86   404    166
87   387    173
88   379    181
89   376    188
90   373    196
91   362    205
92   348    213
93   341    222
94   337    232
95   329    243
96   325    254
97   320    267
98   308    283
99   320    322
100  284    689
```

Now you can look at the N50 stats scaled by the genes with the highest expression. If, for example, you only wanted to see the N50 for the 60% highest expressed genes, it would be ~485.

```
Results of all tutorials can be found here:
/data/genomics/workshops/smsc/RNA_Seq/SMSC_results.tar.gz
```

