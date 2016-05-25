#Combination Connectivity Mapping

`ccmap` finds drugs and drug combinations that are predicted to reverse or
mimic gene expression signatures. These drugs might reverse diseases or mimic 
healthy lifestyles.

**NOTE: required `ccdata` package not yet available. Will be released soon on 
Bioconductor.**

-----------------

####**Query Signatures**


To obtain a query gene expression signature, I reccommend that you perform a 
meta-analysis of all gene expression studies that have compared similar groups.
This approach improves the ranking of the correct drug when the query signature
is derived from independent gene expression data for that same drug (figure 1).
This can be accomplished with the 
[**crossmeta**](https://github.com/alexvpickering/crossmeta) package.

![**Figure 1.** Receiver operating curves comparing query results using 
signatures from individual contrasts (auc = 0.720) to meta-analyses (auc = 0.913).
Queries from signatures generated by meta-analyses for 10 drugs were compared to
queries from the 260 contrasts used in the meta-analyses.](/home/alex/Documents/Batcave/GEO/1-meta/meta-vs-cons.png)

To use `ccmap`, the query signature needs to be a named vector of effect size 
values where the names correspond to uppercase HGNC symbols. If you used 
`crossmeta`, proceeed as follows:

```R
library(crossmeta)

# load previous crossmeta differential expression analysis
anals <- load_diff(gse_names, data_dir)

# run meta-analysis
es <- es_meta(anals)

# extract moderated adjusted standardized effect sizes
dprimes <- get_dprimes(es)

# query signature
query_sig <- dprimes$meta
```

-----------------


####**Drug Signatures**

Drug signatures were generated using the raw data from the [Connectivity Map
build 2](https://www.broadinstitute.org/cmap/). The raw data from experiments with
a shared platform were norm-exp background corrected, quantile normalized, and 
log2 transformed (RMA algorithm). After preprocessing, contrasts were specified
such that all signatures for each drug were compared to all vehicle treated signatures.
Non-treatment related variables (cell-line, drug dose, batch effects, etc.)
were discovered using `sva` and accounted for during differential expression
analysis by `limma`. Finally, moderated t-statistics calculated by `limma` were 
used by `GeneMeta` to calculate moderated unbiased standardised effect sizes.

The final drug signatures are available in the `ccdata` package.

```R
library(ccdata)

# load drug signatures
data(cmap_es)
```


-----------------


####**Querying Drug Signatures**

Genes from query and drug signatures are classified as up or down regulated 
based on the sign of their effect sizes. Overlap between query and drug signatures
are then calculated by determining the number of genes that are regulated in the
same direction. Net overlap is also calculated as the difference between 
overlapping genes and genes regulated in the opposite direction between query and
drug signatures.

The number of query and drug signature genes can be chosen. If so, only the top
most differentially expressed genes are used. Alternatively, a range of query 
sizes can be used and drugs ranked based on the area under the curve formed by
plotting net overlap as a function of query size. This auc metric has the 
advantage of weighting query genes according to their extent of differential 
expression. 

```R
library(ccmap)

# query drug signatures using all common genes
top_drugs <- query_drugs(query_sig)

# query drug signatures using a range of query gene sizes
range_res <- range_query_drugs(query_sig)

```


-----------------


####**Drug Combinations**


To more closely mimic or reverse a gene expression signature, drug combinations
may be promising. For the 1309 drugs in the Connectivity Map build 2, there are 
856086 unique two-drug combinations. It is currently unfeasable to assay all these
combinations, but their expression profiles can be predicted using machine 
learning methods.

In order to do so, a gradient boosted machine was trained using microarray data
from GEO where single treatments and their combinations were assayed. In total,
96 studies with 163 treatment combinations and almost 7.5 million features were 
used for training. The final model is most accurate for transcripts with higher 
absolute standardized effect sizes (figure 2). The model's overall 
classification accuracy is 76.9% (assessed on out-of-fold data). This is a 
modest improvement over some simple benchmark models. For example, predicting 
that the combination treatment effect is an average of the two treatments is 
76.0% accurate. 

![**Figure 2.** Transcripts with higher absolute standardized effect sizes for 
treatment combinations are classified more accurately. A gradient boosted random
forest model in comparison to a a benchmark model where single treatments are
averaged. For reference, about 40% of the data had an absolute effect size 
greater than 1.](/home/alex/Documents/Batcave/GEO/ccdata/data-raw/drug_combos/accuracy.png)

To generate the drug combination signatures database requires substantial 
computing time (approximately 5 days using a Intel® Core™ i7-6700K) and 107GB 
of free memory. To do so proceeed as follows:

```R
library(ccmap)

# make drug combination data base
make_drug_combos()

```


Once the drug combination database is generated, it can be queried as follows:

```R
library(ccmap)

db_dir <- "/path/to/drug_combos.sqlite"

# query drug combination signatures using all common genes
top_combos <- query_combos(query_sig, db_dir)

# query drug signatures using a range of query gene sizes
range_res <- range_query_combos(query_sig, db_dir)

```

  
  
  
