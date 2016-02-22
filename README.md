# AGB_naive_bayes
Project of AGB class: Naive Bayes model to classify tumor types gene expression patterns

INFO AT: http://regulatorygenomics.upf.edu/courses/Master_AGB/Exercise_NaiveBayes/index.html

## TO DO
- [x] Are there prior probabilities to take into account? No, all 1/8.
- [x] Get the sets and parse them.
- [x] Define and create the training sets and the test set (equal n).
- [x] Z-scores to descrete values.
- [x] Measure Likelyhoods.
- [x] Measure Mutual information.
- [x] Do we need pseudocounts? YES

### Get the sets
```bash
mkdir sources

wget http://regulatorygenomics.upf.edu/courses/Master_AGB/Exercise_NaiveBayes/brca_gene_zscore_full-filtered.txt -O ./sources/brca.tbl && \
wget http://regulatorygenomics.upf.edu/courses/Master_AGB/Exercise_NaiveBayes/coad_gene_zscore_full-filtered.txt -O ./sources/coad.tbl && \
wget http://regulatorygenomics.upf.edu/courses/Master_AGB/Exercise_NaiveBayes/hnsc_gene_zscore_full-filtered.txt -O ./sources/hnsc.tbl && \
wget http://regulatorygenomics.upf.edu/courses/Master_AGB/Exercise_NaiveBayes/kirc_gene_zscore_full-filtered.txt -O ./sources/kric.tbl && \
wget http://regulatorygenomics.upf.edu/courses/Master_AGB/Exercise_NaiveBayes/luad_gene_zscore_full-filtered.txt -O ./sources/luad.tbl && \
wget http://regulatorygenomics.upf.edu/courses/Master_AGB/Exercise_NaiveBayes/lusc_gene_zscore_full-filtered.txt -O ./sources/lusc.tbl && \
wget http://regulatorygenomics.upf.edu/courses/Master_AGB/Exercise_NaiveBayes/prad_gene_zscore_full-filtered.txt -O ./sources/prad.tbl && \
wget http://regulatorygenomics.upf.edu/courses/Master_AGB/Exercise_NaiveBayes/thca_gene_zscore_full-filtered.txt -O ./sources/thca.tbl

```
### Look for funy values with AWK
```bash
gawk '{print $3}' coad.tbl | sort | uniq -c | more

gawk '{print $3}' coad.tbl | sort | uniq -c | tail

etc


```
### Filter genes with NA or Inf in at least 1 samples
```bash
cat sources/* | egrep '\bNA\b|\bInf\b' | gawk '{print $1 }' | sort | uniq > not_valid_genes.txt

```
Load them into a hash, and just check in NoSoNaive.

We need to know the minumum number of samples in each cancer type, because it will be the number of samples used in our study.

```bash
gawk '{print FILENAME, NF}' f_* | sort | uniq | more

```
The number is 288.


### Do parser.pl
- [x] Filter train & testing (144 each), where 288 is the minumum number of samples from the cancer type files.
    - WARNING: 144 --> 147 when applying pseudocounts.

- [x] The filter was pseudo-randomized (odds -> test, even -> train). Better against stratification than 0..143 vs 144..end.
- [x] Create a bunch of test and train files.
- [x] Create x2 test and train files sets.
- [ ] Change proportion train vs test.

### Do NotSoNaive.pl
- [x] Filter which genes have at leat 1 value as NA or Inf, and store it within a hash.
    - Filter from the test & training set those genes.
- [x] Calcule Liklihoods.
- [x] Calcule Conditional entropies.
- [x] Calcule entropies.
- [x] Information gain.
- [x] Use directories instead of a bunch of files.
- [ ] Mean between set1 and set2 results.
- [x] Precision recall by cancer.
- [ ] Decide cut-off score, optimizing by precision, maybe recall?

### One command to rule them all

```bash
for int in 0.1 0.15 0.2 0.25 0.3 0.35 0.4 0.45 0.5 0.55 0.6 0.65 0.7 0.75 0.8 0.85 0.9 0.95 1; do \
perl NotSoNaive.pl -dir set1/ -not not_valid_genes.txt -ig 0.5; done

for int in 0.1 0.15 0.2 0.25 0.3 0.35 0.4 0.45 0.5 0.55 0.6 0.65 0.7 0.75 0.8 0.85 0.9 0.95 1; do \
perl -e '$int = shift @ARGV; $pos = 0; $tot = 0; while(<>) {@cols = split /\t/; if ($cols[1] eq $cols[2]) {$pos++; $tot++;} else {$tot++; }} print "$int\t", $pos / $tot, "\n";' "$int" results_${int}.tbl >> tp.tbl; done
```

### Information gain
* 3rd quantile = 0.241287
* Number of genes with IG > 3rd quantile = 4755


### CANCER PLOTS

```bash
perl f_calculator.pl results_from_naive.tbl > f_foreachcancer.tbl
```

```r
library(ggplot2)
fcancer <- read.table(file="f_foreachcancer.tbl", header=T, sep="\t")
ggplot(fcancer) +
    geom_bar(aes(x=MEASURE, y=VALUE, fill=MEASURE), position="dodge", stat="identity") +
    facet_wrap(~ CANCER) +
    theme_bw() +
    theme(axis.ticks.x=element_blank(),axis.text.x=element_blank()) +
    xlab("") + ylab("")
```


### IG PLOT
```bash
perl -e '
%data = ();
while(<>){
    chomp;
    ($ig, $set, $value) = split /\t/;
    $data{$ig}->{$set} = $value;
};
foreach my $ig (keys %data) {
    $mean = 0;
    $sd = 0;
    foreach my $set (keys %{$data{$ig} }) {
        $mean += $data{$ig}->{$set};
    }
    foreach my $set (keys %{$data{$ig} }) {
        $sd += (($data{$ig}->{$set} - ($mean/5)) ** 2)
    }     
    print "$ig\t", $mean / 5, "\t", sqrt($sd/4), "\n"
}' tp.tbl > tp_summary.tbl
```


```r
library(ggplot2)
ig3 <- read.table(file="tp_summary.tbl")

ggplot(ig3, aes(x=V1, y=V2, group=1)) +
    stat_summary(fun.y="mean", geom="line", alpha=0.7) +
    geom_errorbar(aes(ymin=V2-V3, ymax=V2+V3)) +
    stat_summary(fun.y="mean", geom="point", alpha=0.8) +
    theme_bw() +
    ylim(0,1) +
    geom_hline(yintercept=0.125, linetype="dashed", alpha=0.7) +
    xlab("\nI.G. Threshold") + ylab("Precision\n")
```
