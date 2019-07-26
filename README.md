---
title: "RNASEQ_SNV_CALLING"
author: "Yaroslav Ilnytskyy"
date: "July 26, 2019"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

# Preprocess aligned RNA sequencing reads.
1) This workflow starts with single end RNA sequencing reads mapped using HISAT2 to Human genome GRCh37 (Ensembl).
```
python Opossum.py --BamFile <bam> \
                  --OutFile <opossum.bam> \
                  --SoftClipsExist True \
                  --ProperlyPaired False \
                  --MapCutoff 60
```

2) Call variants using Platypus.
```
pythonPlatypus.py callVariants \
       --bamFiles <_opossum.bam> \
       --refFile /illumina/pipeline_in/GENOMES/Homo_sapiens/Ensembl/GRCh37/Sequence/WholeGenomeFasta/genome.fa \
       --filterDuplicates 0 \
       --minHapQual 0 \
       --minFlank 0 \
       --maxReadLength 75 \
       --minGoodQualBases 10 \
       --minBaseQual 20 \
       -o <_platypus.vcf>
```

3) Retain only the variants that pass quality filter (FILTER == 'PASS')
```
vcftools --vcf <platypus.vcf> --remove-filtered-all --recode --recode-INFO-all --out <platypus.recode>
```

4) Retain only the variants that overlap exons. The gtf files with annotation was converted to bed, and parced to contain exons only.
```
vcftools --vcf <platypus.recode.vcf> --recode --recode-INFO-all --bed <exons.bed> --out <platypus.recode.exons.vcf>
```

5) Annotate by gene and predict effect using snpEff.
```
java -Xmx10g -jar ~/programs/snpEff/snpEff.jar GRCh37.75 <platypus.recode.exons.vcf> > <platypus.recode.exons.ann.vcf>
```

6) Annotate with known SNPs from ExAc database and known somatic mutations from COSMIC database. 

ExAc ids.
```
java -jar SnpSift.jar annotate -id ExAC.r1.sites.vep.vcf <platypus.recode.exons.ann.vcf> > <platypus.recode.exons.ann.vcf>
```
```
java -jar ~/programs/snpEff/SnpSift.jar annotate -id CosmicCodingMust.vcf.gz <platypus.recode.exons.exac.vcf> > <platypus.recode.exons.cosmic.vcf>
```

# Software versions
* Opossum 0.2
* Platypus 0.8.1.1-3
* VCFtools 0.1.16
* SnpEff 4.3t
* Bpipe 0.9.9.2

# Annotation sources
1) Exac release 1 variants ftp://ftp.broadinstitute.org/pub/ExAC_release/release1/ExAC.r1.sites.vep.vcf.gz
2) COSMIC somatic mutations dataset: version 89, genome GRCh37, CosmicCodingMust.vcf.gz (this file had to go through bgzip and indexing with tabix)






