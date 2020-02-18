路径：/public/home/lh/tiv16sa2001/MGGZ20191223A1463B-1N__V4_1264samples

### 1. 数据质控

```
cd /public/home/lh/tiv16sa2001/MGGZ20191223A1463B-1N__V4_1264samples/A19012020_V4_1264samples
sz sample-information.xlsx  #将样本的下机缩略信息传输至本地，共收到1265个(共送测1265个)样本数据，其中6个样本未达到要求的3w reads,包括：FVV41727;FVV21902;FVV41833;FVV11902;FVV41711;FVS51840.
mkdir fastqc
mv fastqc sample-information.xlsx ../
fastqc -o ../fastqc -t 32 */*.fq
cd /public/home/lh/tiv16sa2001/MGGZ20191223A1463B-1N__V4_1264samples/fastqc
multiqc *fastqc.zip --ignore *.html
```

###  2. Qiime处理

```
cd /public/home/lh/tiv16sa2001/MGGZ20191223A1463B-1N__V4_1264samples
mkdir v4_1264samples
cd A19012020_V4_1264samples
mv  */* v4_1264samples
cd ../v4_1264samples
rename .fq .fastq *
rename .R _R *
mothur
make.file(inputdir=., type=fastq, prefix=tiv2020)

#创建pbs脚本
vim qiime
#!/bin/bash
#PBS -N qiime
#PBS -l nodes=1:ppn=16
#PBS -j n
#PBS -q batch
#PBS -e ${PBS_JOBNAME}.err
#PBS -o ${PBS_JOBNAME}.out

cd $PBS_0_WORKDIR
source activate qiime2-2019.7
cd /public/home/lh/tiv16sa2001/MGGZ20191223A1463B-1N__V4_1264samples/v4_1264samples

time qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path manifest.txt \
  --output-path demux.qza \
  --input-format PairedEndFastqManifestPhred33V2

time qiime cutadapt trim-paired \
  --i-demultiplexed-sequences demux.qza \
  --p-cores 8 --p-front-f GTGCCAGCMGCCGCGGTAA \
  --p-front-r GGACTACHVGGGTWTCTAAT \
  --o-trimmed-sequences demux_tiv.qza
  
time qiime demux summarize \
  --i-data demux_tiv.qza \
  --o-visualization demux_tiv.qzv
  
time qiime dada2 denoise-paired \
  --i-demultiplexed-seqs demux_tiv.qza \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f 0 \
  --p-trunc-len-r 0 \
  --o-table table.qza \
  --o-representative-sequences rep-seqs.qza \
  --o-denoising-stats denoising-stats.qza
  
time qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences rep-seqs.qza \
  --o-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned.qza \
  --o-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza
  
time qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree.qza \
  --i-table table.qza \
  --p-sampling-depth 10000 \
  --m-metadata-file metadata.txt \
  --output-dir core-metrics-results

##filter mitochondria and chloroplast
time qiime feature-classifier classify-sklearn \
  --i-classifier silva-132-99-515-806-nb-classifier.qza  \
  --i-reads rep-seqs.qza \
  --o-classification taxonomy.qza
  
qiime tools export \
  --input-path taxonomy.qza \
  --output-path taxonomy-with-spaces
  
qiime metadata tabulate \
  --m-input-file taxonomy-with-spaces/taxonomy.tsv \ 
  --o-visualization taxonomy-as-metadata.qzv
  
qiime tools export \
  --input-path taxonomy-as-metadata.qzv \
  --output-path taxonomy-as-metadata
  
qiime tools import \
  --type 'FeatureData[Taxonomy]' \
  --input-path taxonomy-as-metadata/metadata.tsv \
  --output-path taxonomy-without-spaces.qza

time qiime taxa filter-seqs \
  --i-sequences rep-seqs.qza \
  --i-taxonomy taxonomy-without-spaces.qza \
  --p-exclude mitochondria,chloroplast \
  --o-filtered-sequences rep-seqs-no-mitochondria-no-chloroplast.qza

time qiime taxa filter-table \
  --i-table table.qza \
  --i-taxonomy taxonomy-without-spaces.qza \
  --p-exclude mitochondria,chloroplast \
  --o-filtered-table table-no-mitochondria-no-chloroplast.qza
  
time qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences rep-seqs-no-mitochondria-no-chloroplast.qza \
  --o-alignment aligned-rep-seqs-no-mitochondria-no-chloroplast.qza \
  --o-masked-alignment masked-aligned-rep-seqs-no-mitochondria-no-chloroplast.qza \
  --o-tree unrooted-tree-no-mitochondria-no-chloroplast.qza \
  --o-rooted-tree rooted-tree-no-mitochondria-no-chloroplast.qza
  
time qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree-no-mitochondria-no-chloroplast.qza \
  --i-table table-no-mitochondria-no-chloroplast.qza \
  --p-sampling-depth 10000 \
  --m-metadata-file metadata.txt \
  --output-dir core-metrics-results-no-mitochondria-no-chloroplast
  
time qiime feature-classifier classify-sklearn \
  --i-classifier silva-132-99-515-806-nb-classifier.qza  \
  --i-reads rep-seqs-no-mitochondria-no-chloroplast.qza \
  --o-classification taxonomy-no-mitochondria-no-chloroplast.qza
  
```

```
# Plugin error from cutadapt: [Errno 28] No space left on device
# mkdir ./tmp
# export TMPDIR=./tmp
```

```
biom convert -i feature-table.biom -o feature-table.from_biom.txt --to-tsv
```

