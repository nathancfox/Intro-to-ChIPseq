---
title: "ChIP-Seq Quality Assessment"
author: "Mary Piper, Meeta Mistry"
date: "June 12, 2017"
---

Contributors: Mary Piper and Meeta Mistry

Approximate time: 1.5 hours

## Learning Objectives

* Discuss other quality metrics for evaluating ChIP-Seq data
* Generate a report containing quality metrics using the R Bioonductor package `ChIPQC`
* Learn to run R scripts from the command-line
* Identify sources of low quality data


## Additional Quality Metrics for ChIP-seq data

<img src="../img/chip_workflow_june2017_step3.png" width="700">

The [ENCODE consortium](https://genome.ucsc.edu/ENCODE/qualityMetrics.html) analyzes the quality of the data produced using a variety of metrics. We have already discussed metrics related to strand cross-correlation such as NSC and RSC. In this section, we will provide descriptions of additional metrics that **assess the distribution of signal within enriched regions, within/across expected annotations, across the whole genome, and within known artefact regions.**

> **NOTE**: For some of the metrics we give examples of what is considered a 'good measure' indicative of good quality data. Keep in mind that passing this threshold does not automatically mean that an experiment is successful and a values that fall below the threshold does not automatically mean failure!


### SSD

The SSD score is a measure used to indicate evidence of enrichment. It provides a measure of pileup across the genome and is computed by looking at the **standard deviation of signal pile-up along the genome normalised to the total number of reads**. An enriched sample typically has regions of significant pile-up so a higher SSD is more indicative of better enrichment. SSD scores are dependent on the degree of total genome wide signal pile-up and so are sensitive to regions of high signal found with Blacklisted regions as well as genuine ChIP enrichment. 


### FRiP: Fraction of reads in peaks

This value reports the percentage of reads that overlap within called peaks.  This is another good indication of how ”enriched” the sample is, or the success of the immunoprecipitation. It can be considered a **”signal-to-noise” measure of what proportion of the library consists of fragments from binding sites vs. background reads**. FRiP values will vary depending on the protein of interest. A typical good quality TF with successful enrichment would exhibit a FRiP around 5% or higher. A good quality PolII would exhibit a FRiP of 30% or higher. There are also known examples of	good data with FRiP < 1% (i.e. RNAPIII).
	

### Relative Enrichment of Genomic Intervals (REGI)

Using the genomic regions identified as called peaks, we can obtain **genomic annotation to show where reads map in terms of various genomic features**. We then evaluate the relative enrichment across these regions and make note of how this compares to what we expect for enrichment for our protein of interest.


### RiBL: Reads overlapping in Blacklisted Regions

It is important to keep track of and filter artifact regions that tend to show **artificially high signal** (excessive unstructured anomalous reads mapping). As such the DAC Blacklisted Regions track was generated for the ENCODE modENCODE consortia. The blacklisted regions **typically appear uniquely mappable so simple mappability filters do not remove them**. These regions are often found at specific types of repeats such as centromeres, telomeres and satellite repeats. 

<img src="../img/blacklist.png" width="600">

These regions tend to have a very high ratio of multi-mapping to unique mapping reads and high variance in mappability. **The signal from blacklisted regions has been shown to contribute to confound peak callers and fragment length estimation.** The RiBL score then may act as a guide for the level of background signal in a ChIP or input and is found to be correlated with SSD in input samples and the read length cross coverage score in both input and ChIP samples. These regions represent around 0.5% of genome, yet can account for high proportion of total signal (> 10%).

> **How were the 'blacklists compiled?** These blacklists were empirically derived from large compendia of data using a combination of automated heuristics and manual curation. Blacklists were generated for various species including and genome versions including human, mouse, worm and fly. The lists can be [downloaded here.](http://mitra.stanford.edu/kundaje/akundaje/release/blacklists/). For human, they used 80 open chromatin tracks (DNase and FAIRE datasets) and 12 ChIP-seq input/control tracks spanning ~60 cell lines in total. These blacklists are applicable to functional genomic data based on short-read sequencing (20-100bp reads). These are not directly applicable to RNA-seq or any other transcriptome data types. 



## `ChIPQC`: quality metrics report

`ChIPQC` is a Bioconductor package that takes as input BAM files and peak calls to automatically **compute a number of quality metrics and generates a ChIPseq
experiment quality report**. We are going to use this package to generate a report for our Nanog and Pou5f1 samples.

### Setting up 

In order to install the `ChIPQC` package for use on the cluster, we will need to load the R module. Also, you will want to create a directory for your results:

```bash
$ module load gcc/6.2.0 R/3.4.1

$ mkdir ~/chipseq/results/chip_qc/ChIPQC

```

Since installing packages can sometimes be problematic on the cluster, we will not have you do the installation and rather **use the libraries we have prepared for this workshop**. You will likely have this library accessible from the last lesson, but in case you have started a new session on O2, you will need to set the `R_LIBS_USER` environment variable.

```bash
# check if the variable is already set 
$ echo $R_LIBS_USER 

# If the above command returns nothing, then run the command below
$ export R_LIBS_USER="/n/groups/hbctraining/R/library/"

```

The last thing we need is the **sample sheet**. Let's copy it over and take a quick look at it:

```bash
$ cp /n/groups/hbctraining/chip-seq/ChIPQC/samplesheet.csv ~/chipseq/results/chip_qc/ChIPQC

$ less ~/chipseq/results/chip_qc/ChIPQC
```

The **sample sheet** contains metadata information for our dataset.Each row represents a peak set (which in most cases is every ChIP sample) and several columns of required information, which allows us to easily load the associated data in one single command. _NOTE: The column headers have specific names that are expected by ChIPQC!!_. 

* **SampleID**: Identifier string for sample
* **Tissue, Factor, Condition**: Identifier strings for up to three different factors (You will need to have all columns listed. If you don't have infomation, then set values to NA)
* **Replicate**: Replicate number of sample
* **bamReads**: file path for BAM file containing aligned reads for ChIP sample
* **ControlID**: an identifier string for the control sample
* **bamControl**: file path for bam file containing aligned reads for control sample
* **Peaks**: path for file containing peaks for sample
* **PeakCaller**: Identifier string for peak caller used. Possible values include “raw”, “bed”, “narrow”, “macs”
 
### Creating an R script

We are going to create an R script which contains the R code required to generate the report. You can start by using vim to open up the text editor and creating a file called `run_chipQC.R`:

```bash

$ cd ~/chipseq/scripts

$ vim run_chipQC.R
```


**Don't worry about understanding the syntax of the code below, just copy and paste into your text editor.**  There are very few lines of code and so we will briefly explain what each line is doing, so the script is not a complete black box:

Let's start with a shebang line. Note that this is different from that which we used for our bash shell scripts. 

```
#!/usr/bin/env Rscript
```

Next we can load the `ChIPQC` library so we have access to all the package functions and then load the samplesheet into R. 

```
## Load libraries
library(ChIPQC)

## Load sample data
samples <- read.csv('~/chipseq/results/chip_qc/ChIPQC/samplesheet.csv')
```

Next we will create a ChIPQC object. `ChIPQC` will use the samplesheet to read in the data for each sample (BAM files and narrowPeak files) and compute quality metrics. The results will be stored into the object `chipObj`. 

```
## Create ChIPQC object
chipObj <- ChIPQC(samples, annotation="hg19") 
```

The next line of code will export the chipObj from the R environment to an actual physical file. This file can be loaded into R at a later time (on your local computer or on the cluster) and you will have access to all of the metrics that were computed and stored in that object. 

```
## Save the chipObj to file
save(chipObj, file="~/chipseq/results/chip_qc/ChIPQC/chipObj.RData")
```

The final step is taking those quality metrics and summarize information into an HTML report with tables and figures. **We will not run this line of code, and so we have a put `#` sign in front of that line.** 


```
## Create ChIPQC report
# ChIPQCreport(chipObj, reportName="ChIP QC report: Nanog and Pou5f1", reportFolder="~/chipseq/results/chip_qc/ChIPQC/ChIPQCreport")
```

> **NOTE:** The reason we have commented out this line is because in order to generate the report on **O2 you require the X11 system**, which we are currently not setup to do. If you are interested in learning more about using X11 applications you can [find out more on the O2 wiki page](https://wiki.rc.hms.harvard.edu/display/O2/Using+X11+Applications+Remotely). Alternatively, if you are familiar with R you can also copy the `chipObj.RData` object over to your local computer, load it into R and then run the line of code we had commented out.
> 
>  Keep in mind you will have to install the `ChIPQC` package (vChIPQC_1.10.3 or higher) if you are doing this locally. 
> 
```
source("http://bioconductor.org/biocLite.R")
biocLite("ChIPQC")
```

**Your final script should look like this:**


```
#!/usr/bin/env Rscript

## Load libraries
library(ChIPQC)

## Load sample data
samples <- read.csv('~/chipseq/results/chip_qc/ChIPQC/samplesheet.csv')

## Create ChIPQC object
chipObj <- ChIPQC(samples, annotation="hg19")

## Save the chipObj to file
save(chipObj, file="~/chipseq/results/chip_qc/ChIPQC/chipObj.RData")

## Create ChIPQC report
#ChIPQCreport(chipObj, reportName="Nanog_and_Pou5f1", reportFolder="~/chipseq/results/chip_qc/ChIPQC/ChIPQCreport")
```

Save your script and quit `vim`. Now we can run the script interactively. This will take 2-3 minutes to complete and you will see a bunch of text written to the screen as each line of code is run. When completed you can check the `results/chip_qc/ChlPQC` to make sure you have the `.RData` file.

```
$ Rscript run_chipQC.R
```

>**NOTE:** Sometimes the information printed to screen is useful log information. If you wanted to capture all of the verbosity into a file you can run the script using `2>` and specify a file name.
> `Rscript run_chipQC.R 2> ../logs/ChIPQC.Rout`

### `ChIPQC` report

Since our report is based only on a small subset of data, the figures will not be as meaningful. **Take a look at the report generated using the full dataset instead.** Download [this zip archive](https://www.dropbox.com/s/sn8drmjj2tar4xs/ChIPQCreport%20-%20full%20dataset.zip?dl=0). Uncompress it and you should find an html file in the resulting directory. Double click it and it will open in your browser. At the top left you should see a button labeled 'Expand All', click on that to expand all sections.

Let's start with the **QC summary table**:

<img src="../img/QCsummary.png">

Here, we see the metrics mentioned above (SSD, RiP and RiBL). A higher SSD is more indicative of better enrichment. Higher scores are obseved for the Pou5f1 replicates and so greatest enrichment for depth of signal. SSD scores are dependent on the degree of total genome wide signal pile-up and so are sensitive to regions of high signal found with Blacklisted regions as well as genuine ChIP enrichment. The RiBL percentages are not incredibly high (also shown in the plot in the  next section) and FriP percentages are around 5% or higher, except for Pou5f1-rep2. 

Additionally, we see other statistics related to the strand cross-correlation (FragLength and RelCC which is similar to the RSC). With RelCC values larger than one for all ChIP samples suggest good enrichment.

The next table contains **the mapping quality, and duplication rate,** however since we had already filtered our BAM files we find the numbers do not report much for us.

Next is a plot showing the effect of blacklisting, with the proportion of reads that do and do not overlap with blacklisted regions. The final plot in this section uses the genomic annotation to show **where reads map in terms of genomic features**. This is represented as a heatmap showing the enrichment of reads compared to the background levels of the feature. We find that there is most enrichment in promotor regions. This plot is useful when you expect enrichment of specific genomic regions.  
 
<img src="../img/GenomicFeatureEnrichment.png" width="500">

The next section, **ChIP Signal Distribution and Structure**, looks at the inherent ”peakiness” of the samples. The first plot is a **coverage histogram**. The x-axis represents the read pileup height at a basepair position, and the y-axis represents how many positions have this pileup height. This is on a log scale. **A ChIP sample with good enrichment should have a reasonable ”tail”, that is more positions (higher values on the y-axis) having higher sequencing depth**. 
Samples with low enrichment (i.e input), consisting of mostly background reads will have lower genome wide low pile-up. In our dataset, the Nanog samples have quite heavy tails compared to Pou5f1, especially replicate 2. The SSD scores, however, are higher for Pou5f1. When SSD is high but coverage looks low it is possibly due to the presence of large regions of high depth and a flag for blacklisting of genomic regions. The cross-correlation plot which is displayed next is one we have already covered. 

<img src="../img/CoverageHistogramPlot.png" width="500">



The final set of plots, **Peak Profile and ChIP Enrichment**, are based on metric computed using the supplied peaks if available. The first plot shows average peak profiles, centered on the summit (point of highest pileup) for each peak.

<img src="../img/PeakProfile.png" width="500">

The **shape of these profiles can vary depending on what type of mark is being studied** – transcription factor, histone mark, or other DNA-binding protein such as a polymerase – but similar marks usually have a distinctive profile in successful ChIPs. 

Next we have two plots that summarize the number of **Reads in Peaks**. ChIP samples with good enrichment will have a higher proportion of their reads overlapping called peaks. Although RiP is higher in Nanog, the boxplot for the Nanog samples shows quite different distributions between the replicates compared to Pou5f1.

<img src="../img/Rip.png" width="500">

<img src="../img/Rap.png" width="500">


Finally, there are plots to show **how the samples are clustered**. The correlation heatmap is based on correlation values for all the peak scores for each sample. The other plot shows the first two principal component values for each sample. In our dataset, the replicates do cluster by replicate for Pou5f1, which is a positive sign. For Nanog we see that Replicate 1 appears to correlate slightly better with Pou5f1 than with Replicate 2. The PCA also demonstrates distance between the Nanog replicates.

<img src="../img/PeakCorHeatmap.png" width="500">

<img src="../img/PeakPCA.png" width="500">


In general, our data look good. There is some discordance apparent between the Nanog replicates and this combined with lower SSD scores might indicate that while there are many peaks identified it is mainly due to noise. 

## Experimental biases: sources of low quality ChIP-seq data

Once you have identified low quality samples, th next logical step is to troubleshoot what might be causing it.

* **Strength/efficiency and specificity of the immunoprecipitation** 

The quality of a ChIP experiment is ultimately dictated by the specificity of the antibody and the degree of enrichment achieved in the affinity precipitation step [[1]](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3431496/). Antibody deficiencies are of two main types:

1. Poor reactivity against the intended target
2. Non-specific antibody, causing cross-reactivity with other DNA-associated proteins

Antibodies directed against transcription factors must be characterized using both a primary (i.e immunoblot, immunofluorescence) and secondary characterization (i.e knockout of target protein, IP with multiple antibodies).

* **Fragmentation/digestion**

The way in which sonication is carried out can result in different fragment size distributions and, consequently, sample-specific chromatin configuration induced biases. As a result, it is not recommended to use a single input sample as a control for ChIP-seq peak calling if it is not sonicated together with the ChIP sample. 

* **Biases during library preparation:** 

*PCR amplification:* Biases arise because DNA sequence content and length determine the kinetics of annealing and denaturing in each cycle of PCR. The combination of temperature profile, polymerase and buffer used during PCR can therefore lead to differential efficiencies in amplification between different sequences, which could be exacerbated with increasing PCR cycles. This is often manifest as a bias towards GC rich fragments [[2]](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4473780/). **Limited use of PCR amplification is recommended because bias increases with every PCR cycle.**




***

> **NOTE:** Many of the plots that were generated in the ChIPQC report can also be generated using [`deepTools`](http://deeptools.readthedocs.org/en/latest/content/list_of_tools.html), a suite of python tools developed for the efficient analysis of high-throughput sequencing data, such as ChIP-seq, RNA-seq or MNase-seq. If you are interested in learning more we have a [lesson on quality assessment using deepTools](https://github.com/hbctraining/In-depth-NGS-Data-Analysis-Course/blob/may2017/sessionV/lessons/qc_deeptools.md).

***
*This lesson has been developed by members of the teaching team at the [Harvard Chan Bioinformatics Core (HBC)](http://bioinformatics.sph.harvard.edu/). These are open access materials distributed under the terms of the [Creative Commons Attribution license](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0), which permits unrestricted use, distribution, and reproduction in any medium, provided the original author and source are credited.*

*Details on ChIPQC plots was taken from the [ChIPQC vignette](http://bioconductor.org/packages/release/bioc/vignettes/ChIPQC/inst/doc/ChIPQC.pdf), which provides a walkthrough with examples and thorough explanations.*
