#### Honeybee genome resequencing

- Authors: Chao Tong
- Date: Jan-7-2022

- Project description:

#### Qulity control
```
conda install -c bioconda fastqc
conda install -c bioconda trimmomatic
```
```
fastqc {individual id}.fastq
trimmomatic PE *R1_001.fastq.gz *R2_001.fastq.gz -baseout trimmed.fastq LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:35
```
#### Mapping
```
conda install -c bioconda bwa
```
- 1. build genome index
```
bwa index GCF_003254395.2_Amel_HAv3.1_genomic.fna
```
- 2. map reads to honeybee genome to generate sam file
```
bwa mem GCF_003254395.2_Amel_HAv3.1_genomic.fna {individual id}_R1.fq {individual id}_R2.fq > {individual id}.sam
```
- 3. transform sam file to bam file
```
samtools faidx GCF_003254395.2_Amel_HAv3.1_genomic.fna
samtools view -bhS -t GCF_003254395.2_Amel_HAv3.1_genomic.fna.fai -o {individual id}.bam {individual id}.sam
```
- 4. sorting bam file
```
samtools sort {individual id}.bam -o {individual id}.sorted.bam
```
- 5. build index for bam file
```
samtools index {individual id}.sorted.bam
```
- 6. estimate the depth of sequencing
```
samtools depth {individual id}.sorted.bam > {individual id}_depth.txt
```
- 7. mapping summary
```
samtools flagstat {individual id}.sorted.bam > {individual id}_counts.txt
```

#### Prepare Genotype and Phenotype file for GEMMA

- 1. remove PCR duplication
```
samtools rmdup {individual id}.sorted.bam {individual id}_nopcr.bam
```

- 2. call snp

```
samtools mpileup -g -f <ReferenceGenome> <All file names> | 
bcftools call --skip-variants indels --variants-only -mv -Oz > output.vcf.gz
```

- 3. extract dosage and genotype likelihoods

```
bcftools +dosage output.vcf -- -t GL
bcftools +dosage output.vcf > output.dosage.txt
```

- 4. create relatedness matrix
```
gemmma -g Genotype.txt -p Phenotype.txt -gk 1 -o Relatedness.txt
```

- 5. GEMMA run
```
gemmma -g Genotype.txt -p Phenotype.txt -n <num of trait> -k Relatedness.txt -lm 4 -o Results.txt
```

#### Detect variant (optional)
- 1. remove PCR duplication
```
samtools rmdup {individual id}.sorted.bam {individual id}_nopcr.bam
```
- 2. generate bcf file
```
samtools mpileup -gf GCF_003254395.2_Amel_HAv3.1_genomic.fna {individual id}_nopcr.bam > {individual id}.bcf
```
- 3. variantion detection
```
bcftools call -vm {individual id}.bcf -o {individual id}.variants.bcf
bcftools view -v snps,indels {individual id}.variants.bcf > {individual id}.snps.vcf
```
- 4. filtering variant site
```
bcftools filter -o {individual id}.snps.filtered.vcf -i 'QUAL>20 &&DP>5' {individual id}.snps.vcf
```



#### Annotate variant
- 1. generate annovar input file
download and install **Annovar** [link](http://www.openbioinformatics.org/annovar/download/0wgxR2rIVP/annovar.latest.tar.gz)
```
convert2annovar.pl -format vcf4 {individual id}.snps.vcf > {individual id}.snps.avinput
```
- 2. build local bee database
```
conda install -c bioconda ucsc-gff3togenepred
```
```
gff3ToGenePred -useName GCF_003254395.2_Amel_HAv3.1_genomic.gff genome_refGene.txt
```
```
cut -f 12 genome_refGene.txt > column1.txt
cut -f 2-15 genome_refGene.txt > column_else.txt
paste column1.txt column_else.txt > genome_new_refGene.txt
```
```
retrieve_seq_from_fasta.pl -format refGene -seqfile GCF_003254395.2_Amel_HAv3.1_genomic.fna -outfile genome_new_refGeneMrna.fa genome_new_refGene.txt
cp genome_new_refGene* ./beedb/
```
```
annotate_variation.pl --geneanno --dbtype refGene --buildver genome_new  {individual id}.snps.avinput ./beedb/
mv *_function result
```


