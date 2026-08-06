# Metagenome assembly pipeline

### Requierements
Nextflow >=24.10.0 with Java.   
Singularity/Appteiner.  
Python 3.  
Input FASTQ and host-reference files visible from the compute nodes.  

### Minimal local execution
```
nextflow run main.nf \
-profile singularity \
--input samplesheet.csv \
--outdir results 
```
#### Sample sheet
Relative paths to FASTQ samples and host reference FASTA are resolved in the sample sheet.  
The pipeline accepts a comma-separated CSV supplied through --input.  
Columns of tje sample sheet  
sample: unique sample ID
fastq_1: R1 FASTQ (.fq, .fastq, .fq.gz, .fastq.gz)
fastq_2: R2 FASTQ
host_id: Host reference ID
host_fasta: Host genome FASTA path (FASTA/FA/FNA suffixes, optionally gzip-compressed.)

### Commands
Read QC  
Tool and container  
Tool: FastQC 0.12.1  
Container parameter: container_fastqc  
Default image: docker://staphb/fastqc:0.12.1 

```
fastqc --quiet --threads <cpus> R1.fastq.gz R2.fastq.gz
```

Read filtering
Tool and container  
Tool: fastp 1.3.3  
Default image: docker://staphb/fastp:1.3.3    

```
fastp \
--in1 R1.fastq.gz \
--in2 R2.fastq.gz \
--out1 SAMPLE.trimmed_R1.fastq.gz \
--out2 SAMPLE.trimmed_R2.fastq.gz \
--detect_adapter_for_pe \
--qualified_quality_phred 20 \
--unqualified_percent_limit 30 \
--length_required 50 \
--n_base_limit 5 \
--trim_poly_g \
--thread <cpus> \
--json SAMPLE.fastp.json \
--html SAMPLE.fastp.html
```

Host reference index and read alignment against reference    
Tool and container  
Bowtie2 2.5.5  
Default image: docker://staphb/bowtie2:2.5.5  

```
bowtie2-build \
--threads <cpus> \
host_reference.fa.gz \
host_id
```
```
bowtie2 \
--very-sensitive \
--threads <cpus> \
--met-file SAMPLE.host.bowtie2.metrics.tsv \
-x HOST_ID \
-1 SAMPLE.trimmed_R1.fastq.gz \
-2 SAMPLE.trimmed_R2.fastq.gz \
2> SAMPLE.host.bowtie2.log \
| gzip -1 -c > SAMPLE.host.sam.gz
```
Host decontamination
Tool and container  
SAMtools 1.23  
Default image: docker://staphb/samtools:1.23  

```
samtools view -f 12 -F 2304
```

Metagenome assembly    
Tool and container       
MEGAHIT 1.2.9    
Default image: docker://quay.io/biocontainers/megahit:1.2.9--h8b12597_0     

```
megahit \
--presets meta-sensitive \
--min-contig-len 200 \
--num-cpu-threads <cpus> \
--memory 0.90 \
-1 SAMPLE.host_removed_R1.fastq.gz \
-2 SAMPLE.host_removed_R2.fastq.gz \
-o SAMPLE.megahit_out
``` 

Contig filtering
Tool and container    
SeqKit 2.13.0    
Default image: docker://staphb/seqkit:2.13.0    

```
seqkit seq \
--min-len LENGTH \
--line-width 0 \
SAMPLE.final.contigs.fa \
> SAMPLE.contigs.minLENGTH.fa
```
Assembly QC    
Tool and container    
QUAST 5.3.0    
Default image: docker://staphb/quast:5.3.0    

```
quast.py \
SAMPLE.final.contigs.fa \
SAMPLE.contigs.min1000.fa \
--labels all_contigs,min1000 \
--threads <cpus> \
--min-contig 500 \
--contig-thresholds 0,500,1000,1500,5000,10000,25000,50000 \
--output-dir SAMPLE.quast
```



