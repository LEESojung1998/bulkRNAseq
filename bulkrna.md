# 🧬 Bulk RNA-seq Pipeline (Full Script)

This script runs a complete RNA-seq workflow from FASTQ to gene counts.

---

## 🔧 Requirements

Make sure the following tools are installed:
- fastqc
- cutadapt
- STAR
- samtools
- featureCounts

## 📜 Full Bash Script

```bash
# USER INPUTS
THREADS=8
GENOME_FA="genome.fa"
GTF="annotation.gtf"
STAR_INDEX="star_index"
ADAPTER="AGATCGGAAGAGC"

R1="sample_R1.fastq.gz"
R2="sample_R2.fastq.gz"
SAMPLE="sample"

# STEP 1: QC
echo "Running FastQC..."
fastqc ${R1} ${R2}

# STEP 2: Adapter Trimming
echo "Running Cutadapt..."
cutadapt \
  -a ${ADAPTER} \
  -A ${ADAPTER} \
  -q 20 \
  -m 20 \
  -o ${SAMPLE}_R1.trimmed.fastq.gz \
  -p ${SAMPLE}_R2.trimmed.fastq.gz \
  ${R1} ${R2}

# STEP 3: STAR Index (run once)
if [ ! -d "${STAR_INDEX}" ]; then
  echo "Building STAR index..."
  mkdir -p ${STAR_INDEX}
  STAR --runThreadN ${THREADS} \
       --runMode genomeGenerate \
       --genomeDir ${STAR_INDEX} \
       --genomeFastaFiles ${GENOME_FA} \
       --sjdbGTFfile ${GTF} \
       --sjdbOverhang 100
fi

# STEP 4: Alignment
echo "Running STAR alignment..."
STAR --runThreadN ${THREADS} \
     --genomeDir ${STAR_INDEX} \
     --readFilesIn ${SAMPLE}_R1.trimmed.fastq.gz ${SAMPLE}_R2.trimmed.fastq.gz \
     --readFilesCommand zcat \
     --outFileNamePrefix ${SAMPLE}_ \
     --outSAMtype BAM SortedByCoordinate

BAM="${SAMPLE}_Aligned.sortedByCoord.out.bam"

# STEP 5: Post-alignment QC
echo "Running samtools flagstat..."
samtools flagstat ${BAM} > ${SAMPLE}_flagstat.txt

# STEP 6: Gene Quantification
echo "Running featureCounts..."
featureCounts -T ${THREADS} \
  -a ${GTF} \
  -o ${SAMPLE}_counts.txt \
  -p \
  -s 0 \
  ${BAM}

echo "Pipeline complete!"