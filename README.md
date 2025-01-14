# Manual:

The included Python and R scripts are used to infer TE clades from copy number data as detailed in https://doi.org/10.1093/molbev/msab336. Detailed Jupyter and Rmarkdown notebooks will walk you through implementing the modules, but this README will include brief overview of usage of the packages. 

## Necessary python libraries:
    pandas, numpy, os, seaborn, matplotlib, sys, scipy, scikit-learn, biopython, pysftp, datetime, statsmodels, functools, multiprocessing, pysam

## Necessary R libraries:
    pheatmap, plyr, ggplot2, ggdendro, matrixStats, RColorBrewer, reticulate, viridis, igraph, Polychrome

## Test data:
Test data in this repository is derived from 85 D. melanogaster genomes from the Global Diversity Lines (https://www.g3journal.org/content/5/4/593). These data have been used to infer clades in 41 recently active TEs, and the notebooks included in this repository will allow the user to reproduce clade inferences from (https://www.biorxiv.org/content/10.1101/2021.04.29.442051v1). 

## Aligning data and producing copy number matrices:

Short-read data (https://www.g3journal.org/content/5/4/593) was aligned using ConTExt (https://github.com/LaptopBiologist/ConTExt) and associated scripts and methods were used to generate copy number and SNP pileup files (https://academic.oup.com/genetics/article-abstract/217/2/iyaa027/6043924). Although we recommend using these methods to generate copy number data for clade inference it is not strictly necessary. Any method that produces allele frequencies from alignments and can estimate copy number from read depth of alignments to TE consensus sequences would be adequate.

It is necessary that the **allele copy number** data for the clade inference pipeline be formatted as a **numpy** file with the following dimensions S x n+1 x 4. Where S is the number of individuals in the data set, and n is the number of basepairs in the TE consensus. Each element in the 1st dimension of the matrix corresponds to an individual from your dataset. Each element of the 2nd dimension corresponds to a particular basepair position in the TE sequence plus a placeholder at the first position which should be filled with zeros. Each element in the third dimension is a nucleotide in the order [A, T, C, G]. Thus this matrix will tell us for each individual in our dataset what the copy number is at every position and for each possible nucleotide at that position. Our clade inference pipeline requires a matrix formatted in this way to function.

## Clade inference:

The clade inference tool is broken up into two parts: 1. A python script (haploTE.py) that will filter the allele copy number numpy data and output simplified CSVs with the copy number information. 2. An R script that takes in the copy number CSVs and outputs the clade calls. 

### Necessary files:
Three files are ncessary for this tool: 
1. An **allele copy number numpy** file for each TE (described above)
2. A **sample sheet** for your genomics data set
3. A **TE list** of the names of transposons being analyzed. 

The **sample sheet** file is a two column comma seperated file with an entry for each individual in your genomics dataset. The first column is the population of origin of an individual and the second column is the sample ID of that individual. This is necessary for computing population level summary statistics for both pre-processing data, and for the final output files. 

You also need a **TE list** of the designations for each TE you wish to analyze. This file should also be two columns that are comma seperated. The first column is the name of your TE (which should match the numpy file names), and the second column should contain class/subclass information e.g. LTR, LINE, etc. (second column is not necessary for clade inference and may be filled with placeholders). 

### 1) Generate copy number input:

(NOTE: Future versions will optimize this process)

First extract the minor alleles from the copy number numpy file to a CSVs by using the haploTE.py modules. An example of this implementation is described in cladeInference1.ipynb. 

#### Input:
The haploTE.py module requires an **allele copy number numpy file**, **sample sheet** and **TE list** as described previously. 

#### Parameters:
The user must define allele filtering parameters: **minimum positional sequence diversity**, **minimum allele population frequency**, **minimum allele copy number**, and **minimum number of strains with an allele**. The default parameters for these filters are: 0.1, 0.1, 0.5 and 10, respectively. But the optimal value for the user may depend on the TEs being analyzed, the number of samples, and organism. 


#### Output:
This module will return an S x N matrix for each TE, where S is the number of strains and N is the number of alleles. Each cell in the matrix represent the copy number of that allele in that strain. This table will be fed into the Subfamily_inference.R module where the clade inference will occur. 


### 2) Implement hierarchical clustering:

Once the allele copy number CSVs have been generated use the Subfamily_inference.R module to implement hierarchical clustering to infer clades and generate summary statistics of each clade. The cladeInference.Rmd notebook will run you through an example implentation using the test data in the repository. 

#### Input:
Subfamily_inference.R requires an **allele copy number CSV** for each TE to infer clades as well as the **allele copy number numpy** files to generate population level summary statistics. Also necessary is the **population sample sheet** and the **TE list**. You may require an alternate **allele copy number CSV** with individuals that are extreme **outliers** in TE copy number removed to perform initial inference, but this is optional. 

#### Parameters:
The user must input a **distance cut-off** for hierarchical clustering. The inference algorithm builds a correlation matrix and converts this matrix into a distance matrix by computing the dissimilarity distance (1 - correlation coefficient). Distance cut-off must therefore be in input as 1 minus the desired correlation cut-off. Determining the optimal distance cut-off is non-trivial and we recommend a procedure similar to one outlined in (https://doi.org/10.1093/molbev/msab336) to determine a satisfactory cut-off. Ultimately, there is coarse-graining in this procedure and there will be no single parameter that optimally clusters alleles. You may additionally desire to change the linkage method in the clustering algorithm, which can be re-defined by any linkage method from R's pheatmap function. Importantly, you must define a color palette for each of your populations for visualization.

#### Output:
The Subfamily_inference.R module will produce several outputs, both graphical and tabular for each TE in a new directory:

1. Firstly, you will receive a visualization of the seriated correlation **heatmap** for each TE (.haplotype.hmap.png) showing the degree of correlation between pairwise combinations of alleles. Along with this is a **dendrogram** (.dendro.png), visualizing the hierarchical clustering of the alleles. These are useful for qualitatively assessing clustering cut-off accuracy. The program will produce an additional seriated **heatmap** (.pop.hmap.png) showing the copy number of each allele in each strain labelled by their respective populations, which can be useful for visualizing population structure.
2. The tabular outputs you will receive are a .haplotypeTable.tsv and \_cluster_CN.tsv. The .haplotypeTable.tsv is a tab delimited file that contains all relevant summary statistics and copy number of each clade. Each row corresponds to a clade and has a unique Cluster_ID. The first few columns of the table will retain population level summary statistics, such as average population copy number and population frequency of each clade. Following an empty column the remaining columns contain the copy number of each clade in the individuals in the dataset. The \cluster_CN.tsv is a redundant table and is depreciated.


Using the test data and the notebooks presented in this repository you will be able to reproduce our clade inferences from (https://doi.org/10.1093/molbev/msab336). These modules are generalizable enough to be used in any population genomics data set.
