# Basic of RNA-seq analysis

## Environment setup
```
mamba create -n rnaseq python=3.10
mamba activate rnaseq
```

## Installtion of necessary tools
```
mamba install -c bioconda parallel-fastq-dump -y
mamba install -c bioconda trim-galore -y
mamba install -c bioconda fastqc -y
mamba install -c bioconda hisat2 -y
mamba install -c bioconda samtools -y
mamba install -c bioconda stringtie -y
mamba install -c bioconda bioconductor-deseq2 -y
mamba install -c anaconda jupyter -y
mamba install -c anaconda pandas -y
mamba install -c conda-forge matplotlib -y
mamba install -c conda-forge nbclassic -y
mamba install -c r r-irkernel -y
```

## Reference file preparation
Download all the necessary reference files from NCBI database (takes approximately 20min)
https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_000001405.40/
```
curl -OJX GET "https://api.ncbi.nlm.nih.gov/datasets/v2alpha/genome/accession/GCF_000001405.40/download?include_annotation_type=GENOME_FASTA,GENOME_GFF,RNA_FASTA,CDS_FASTA,PROT_FASTA,SEQUENCE_REPORT&filename=GCF_000001405.40.zip" -H "Accept: application/zip"
unzip GCF_000001405.40.zip "ncbi_dataset/data/GCF_000001405.40/*" -d ref
rm GCF_000001405.40.zip
```

## Genome index file
Indexing genome file for HISAT2
```
mkdir index
hisat2-build -p 20 ref/ncbi_dataset/data/GCF_000001405.40/GCF_000001405.40_GRCh38.p14_genomic.fna index/human_genome
```

## Example fastq files
Download public RNA-seq data  
https://trace.ncbi.nlm.nih.gov/Traces/?view=study&acc=SRP252863  
```
parallel-fastq-dump --sra-id SRR11309003 --threads 4 --outdir fastq --split-files --gzip
parallel-fastq-dump --sra-id SRR11309004 --threads 4 --outdir fastq --split-files --gzip
parallel-fastq-dump --sra-id SRR11309005 --threads 4 --outdir fastq --split-files --gzip
parallel-fastq-dump --sra-id SRR11309006 --threads 4 --outdir fastq --split-files --gzip
```
HEK 293 cell: SRR11309003 SRR11309004  
HBEC 5i cell: SRR11309005 SRR11309006  

## Output directories  
Creation of directories for output data
```
for i in SRR11309003 SRR11309004 SRR11309005 SRR11309006
do
mkdir -p analysis/$i
done
```

## RNA-seq workflow  
<img src="fig/RNAseqWorkflow.png" width='300'>

## 1️⃣ Step 1 : Processing fastq in each sample directory  (SRR11309003 as an example)
### Move to the working derectory
```
cd analysis/SRR11309003
```

### Quality chceck
[fastqc](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)  
```
mkdir qc
fastqc -t 4 -o qc ../../fastq/SRR11309003_1.fastq.gz ../../fastq/SRR11309003_2.fastq.gz
```

### Adaptor trimming
[trim-galore](https://github.com/FelixKrueger/TrimGalore/blob/master/Docs/Trim_Galore_User_Guide.md)
```
mkdir trimmed_fastq
trim_galore -j 20 --paired ../../fastq/SRR11309003_1.fastq.gz ../../fastq/SRR11309003_2.fastq.gz -o trimmed_fastq
```

### Alignment
[HISAT2](https://daehwankimlab.github.io/hisat2/manual/)  
```
hisat2 -p 16 -x ../../index/human_genome -1 trimmed_fastq/SRR11309003_1_val_1.fq.gz -2 trimmed_fastq/SRR11309003_2_val_2.fq.gz -S SRR11309003.sam 
```

### SAM and BAM file processing
[samtools](https://www.htslib.org/doc/samtools.html)
```
samtools sort SRR11309003.sam -@ 10 -O bam -o SRR11309003.sort.bam 
samtools index SRR11309003.sort.bam
rm SRR11309003.sam
```

### Read count
[StringTie](https://ccb.jhu.edu/software/stringtie/index.shtml?t=manual)
```
stringtie -p 10 -e -G ../../ref/ncbi_dataset/data/GCF_000001405.40/genomic.gff -o SRR11309003.gtf -A  SRR11309003.table SRR11309003.sort.bam
```

## 2️⃣ Step 2: Integration and analysis of the results in Step 1

### Move to analysis/ directory
Go to analysis/ directory where subdirectories of each sample are located  
<img src="fig/Tree.png" width='300'>

### Create read count table (gene_count_matrix.csv) for DESeq2
[prepDE.py3](https://ccb.jhu.edu/software/stringtie/index.shtml?t=manual#deseq)
```
wget -c https://ccb.jhu.edu/software/stringtie/dl/prepDE.py3
chmod +x prepDE.py3
python ./prepDE.py
```

### DEG(differentially expressed genes) analysis
[DESeq2(R)](https://github.com/thelovelab/DESeq2)  
**ipynb**: analysis/DESeq2.ipynb
```
# Variable settings
in_f <- "gene_count_matrix.csv"        # input readcount data
out_f1 <- "results/result.txt"        # output file for the result
out_f2 <- "results/result.png"        # output image file for M-A plot
param_G1 <- 2        # sample number of group 1
param_G2 <- 2        # sample number of group2 
param_FDR <- 0.05        # false discovery rate (FDR) threshold
param_fig <- c(400, 380)        # figure size of M-A plot

# Loading necessary R package
library(DESeq2)        # loading DESeq2 package

# Loading  readc ount data
data <- read.table(in_f, header=TRUE, row.names=1, sep=",", quote="")

# Preprocessing
data.cl <- c(rep(1, param_G1), rep(2, param_G2))
colData <- data.frame(condition=as.factor(data.cl))
d <- DESeqDataSetFromMatrix(countData=data, colData=colData, design=~condition)

# Defferentially expressed genes analysis (DESeq2)
d <- DESeq(d)                          # DESeq2 excution
tmp <- results(d)                      # assign the result 
p.value <- tmp$pvalue                  # assign the p-value
p.value[is.na(p.value)] <- 1           # replace NA by 1
q.value <- tmp$padj                    # assgin the adjusted p-value in q.value
q.value[is.na(q.value)] <- 1           # replace NA by 1
ranking <- rank(p.value)               # assign the ranking based on p-value
log2.FC <- tmp$log2FoldChange        # assign the log2FC
sum(q.value < param_FDR)               # the number of genes (q.value < param_FDR)
sum(p.adjust(p.value, method="BH") < param_FDR)        # the number of genes (q.value < param_FDR, BH-method)

# saving the result into the output file
tmp <- cbind(rownames(data), data, p.value, q.value, ranking, log2.FC)
write.table(tmp, out_f1, sep="\t", append=F, quote=F, row.names=F) 

# saving MA-plot
png(out_f2, pointsize=13, width=param_fig[1], height=param_fig[2])
plotMA(d)
```

## Overview of the result
**ipynb**: analysis/result_overview.ipynb
```
# importing pandas library
import pandas as pd

# Loading the DESeq2 result file 
df = pd.read_csv('results/result.txt', sep='\t')

# Sorting by "ranking"
df = df.sort_values('ranking')

# Showing the dataframe
display(df)
```
