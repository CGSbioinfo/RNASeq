# RnaSeq pipeline

## Getting Started

1. Create a new folder in fs3 or any other location with the project name - called the "project folder" in this README.   
2. Create a bin/ folder in the project folder.   
3. Download and copy the scripts to the bin/ folder.   

## Running the pipeline

### 1. Analysis info file
A central part of the pipeline is the **analysis info** file. It has information about the project location, the original run folder, the reference fasta, gtf and bed files, and the parameters used throughout the analysis.   

#### Format of the analysis info file
The analysis info file is a simple .txt file, with each line providing information. Parameters are separated by the semicolon (i.e ";") character.   

The following is an example of the analysis info file:   

--------------------------------------------------------------------------------------------------   
**working_directory =** /mnt/cgs-fs3/Sequencing/Pipelines/RNASeq_pipeline/example_small_files/    
**run_folder =** /mnt/cgs-fs3/Sequencing/Pipelines/RNASeq_pipeline/example_small_files/xxx   
**run_samplesheet =** /mnt/cgs-fs3/Sequencing/Pipelines/RNASeq_pipeline/example_small_files/xxx    
**bcl2fastq_output =** /mnt/cgs-fs3/Sequencing/Pipelines/RNASeq_pipeline/example_small_files/fastq/   
**readType =** pairedEnd    
**reference_genome =**    
**trimgalore_params =** --gzip; --paired; --fastqc; --fastqc_args '--nogroup --extract'   
**mapping_params =** --runThreadN 4; --outSAMtype BAM SortedByCoordinate; readFilesCommand zcat   
**ncores =** 8   
--------------------------------------------------------------------------------------------------   

The following is the explanation of the analysis info file:   

--------------------------------------------------------------------------------------------------   
**working_directory =** *\<path to directory of the analysis\>*   
**run_folder =** *\<path to the run folder\>*    
**run_samplesheet =** *\<sample sheet to be used to generate fastq files. This is created using the Illumina Expert Manager\>*    
**bcl2fastq_output =** *\<path to output of bcl2fastq. The defaults is fastq/ and the folder will be created automatically\>*    
**readType =** *\<either pairedEnd or singleEnd\>*     
**reference_genome =** *\<path to the STAR reference genome folder that will be used at the mapping step\>*   
**trimgalore_params =** *\<parameters to be passed to trim galore\>* 
**mapping_params =** *\<parameters to be passed to star\>*
**ncores =** *\<Number of cores to use to pararellize analysis\>*    
--------------------------------------------------------------------------------------------------    
   
<br>   

#### Reference genome   
The reference genome should be the output of STAR's *genomeGenerate* command. We currently download genomes from [ensembl](ftp://ftp.ensembl.org/pub/).      

#### How to create the analysis info file

Use the bin/analysis_info.txt.   


```bash
$ python bin/analysis_info.py
```

This will create a file analysis_info.txt, which you can open in a text editor and fill.

<br>   

### 2. Obtain FASTQ files

#### Using bcl2fastq
**Script:** bin/run_bcl2fastq.py   

**Input:** the analysis_info.txt file, which has information about the **run folder**, the **run samplesheet**, and the **bcl2fastq_output** folder. If you have more than one run, you can run this command for each run by changing the run\_folder and run\_samplesheet fields in the analysis\_info.txt file).   

**Command:**   

```bash
$ python bin/run_bcl2fastq.py --analysis_info_file analysis_info.py   
```

**Output:** fastq/ folder, which has a project subfolder with the fastq.gz files, among other files and subfolders.  


#### Downloading data from basespace (Currently used for Lexogen projects)

For lexogen projects, we currently download the data from Basespace. Normally, the downloaded data will have the following structure:

-   A main folder with the project name
-   Subfolders, which correspond to each of the sequenced samples
-   Within each subfolder, four fastq.gz files from the corresponding sample

The tasks done are:

-   Unzip the fastq files:
    -   Move to the downloaded folder.
    -   Use the following commands:


```bash
$ ls -d * | parallel -j 4 --no-notice "cd {} ; gzip -d *"      
```

-   Change the folder name of each sample to a string consisting of **the sample name, an underscore, a capital S, and one or more numbers (eg. sample1\_S1)**. You can use the following commands, however **make sure that the sample names consists of the original sample name, an underscore, a capital S, and one or more numbers (eg. sample1\_S1)**:
    -   Check what the outcome would be. This command will print the original folder name, followed by a space and the new folder name:

```bash
$ for i in $(ls -d *); do sample_name=$(ls ${i} | sed 's/.*\///g' | sed 's/_L00[[:digit:]]\+.*//g'  | sort | uniq); echo ${i} ${sample_name}; done         
```
    -   If it looks correct, use mv to change the folder names:    

```bash
$ for i in $(ls -d *); do sample_name=$(ls ${i} | sed 's/.*\///g' | sed 's/_L00[[:digit:]]\+.*//g'  | sort | uniq); mv ${i} ${sample_name}; done     
```

-   For each sample, concatenate the four files into one:    
    -   The following command tests whether the outcome of the command is right. It prints the sample name, it's four fasta files, and the name of the output concatenated file.    

```bash
$ ls -d * | parallel -j 4 --no-notice "echo {}; ls {}/{}*L001_R1*  {}/{}*L002_R1* {}/{}*L003_R1*  {}/{}*L004_R1*; echo {}/{}_R1_001.fastq"     
```
    -   If that looks correct, concatenate the files for R1 reads:   

```bash
$ ls -d * | parallel -j 4 --no-notice "cat {}/{}*L001_R1*  {}/{}*L002_R1* {}/{}*L003_R1*  {}/{}*L004_R1* > {}/{}_R1_001.fastq"     
```
    -   And for R2 reads for paired end reads:    

```bash
$ ls -d * | parallel -j 4 --no-notice "cat {}/{}*L001_R2*  {}/{}*L002_R2* {}/{}*L003_R2*  {}/{}*L004_R2* > {}/{}_R2_001.fastq"    
```

<br>
    
### 3. Move the reads to a new folder named rawReads    
Use mv command in bash.   

<br>   

### 4. Create sample names file   
**Bash command:**      

```bash
$ ls rawReads | sed 's/_R1_001.fastq//g' | sed 's/_R2_001.fastq//g' | sort | uniq > sample_names.txt   
```

<br>   

### 5. Quality control of Fastq files
To run fastqc in all the samples, use the script bin/qcReads.py.

**Script:** bin/qcReads.py

**Arguments:**   

--------------------------------------------------------------------------------------------------
-h, --help	show this help message and exit
-v, --version	show program's version number and exit
--analysis_info_file	ANALYSIS_INFO_FILE. Text file with details of the analysis. Default=analysis_info.txt
--in_dir	IN_DIR. Path to folder containing fastq files. Default=rawReads/
--out_dir	OUT_DIR. Path to out put folder. Default=rawReads/
--sample_names_file	SAMPLE_NAMES_FILE. Text file with sample names to run. Default=sample_names.txt
--------------------------------------------------------------------------------------------------

**Output:**
Fastqc files and folders will be created for each sample in the rawReads/ folder

**Command example:**


```bash
$ python bin/qcReads.py --analysis_info_file analysis_info.txt --in_dir rawReads/ --out_dir rawReads/ --sample_names_file sample_names.txt
```

<br>   

### 6. Table and plot of number of reads per sample

**Script:** bin/indexQC.R

**Requires:** ggplot2 and reshape R libraries

**Input:** The input directory should have the *fastqc\_data.txt* files of all samples of the project. The 7th line in the fastqc\_data.txt files have the total number of sequences. This is the information that the script collects.

**Command:**   


```bash
$ /usr/bin/Rscript bin/indexQX.R <input dir> <output dir>
```

**Command example:**   


```bash
$ /usr/bin/Rscript bin/indexQX.R rawReads/ Report/figure/data/
```

**Output:** The output directory will have a csv file with the Total number of Sequences and a bar plot with the proportion of reads represented by each sample from the total number of reads from the run.

<br>   

### 7. FastQC plots 

**main script:** bin/fastqc_tables_and_plots.py    

**sub scripts:** bin/create_fastqcPlots_allSamples.R; bin/create_fastqcPlots_perSample.R; bin/create_fastqcTables.py   

**Arguments main script:**     

--------------------------------------------------------------------------------------------------
-h, --help   show this help message and exit
-v, --version        show program's version number and exit
--in_dir IN_DIR       Path to folder containing fastq files. Default=rawReads/
--out_dir_report OUT_DIR_REPORT Path to out put folder. Default=Report/figure
--suffix_name SUFFIX_NAME Suffix to optionally put to the output name. Default=\'\'
--sample_names_file SAMPLE_NAMES_FILE Text file with sample names. Default=sample_names.txt
--plot_device PLOT_DEVICE Specify the format of the plot output. Default=png
--------------------------------------------------------------------------------------------------


**Command example:**      

```bash
$ python bin/fastqc_tables_and_plots.py --in_dir rawReads/ --out_dir_report Report/figure/rawQC/ --suffix_name _raw --sample_names_file sample_names.txt 
```

<br>   

### 8. Trim low quality bases and adapters   

**main script:** bin/trimmingReads.py   

**sub script:** bin/trimming_summary.R     

**analysis info file:** Define the trimgalore parameters that you want to pass to trim galore and the number of cores.      

**Lexogen projects analysis info: ** For lexogen projects, the argument *--clip_R1 12* is added in the analysis_info.txt file (trimgalore_params line) to remove the first 12 bases as recommended. 

**Arguments:**   

--------------------------------------------------------------------------------------------------
-h, --help   show this help message and exit
-v, --version        show program's version number and exit
--analysis_info_file ANALYSIS_INFO_FILE. Text file with details of the analysis. Default=analysis_info.txt
--in_dir     IN_DIR. Path to folder containing fastq files. Default=rawReads/
--out_dir    OUT_DIR. Path to out put folder. Default=trimmedReads/
--sample_names_file  SAMPLE_NAMES_FILE. Text file with sample names to run. Default=sample_names.txt
--------------------------------------------------------------------------------------------------

**Command example:**      

```bash
$ python bin/trimmingReads.py
```

<br>   

### 9. Trimming summary   

The script bin/trimming_summary creates a table with the number of raw sequences, the number of sequences after trimming, and the percentage of sequences removed after trimming.

It expects an input directory of raw data with a fastqc output file for each sample (fastqc_data.txt).   

It also expects an input directory or trimmed data with a fastqc output file for each sample (fastqc_data.txt).   
   
**Arguments:**   

```bash
/usr/bin/Rscript bin/trimming_summary.R <raw_data_indir> <trimmed_data_indir> <outdir>   
```

**Command example:**   

```bash
/usr/bin/Rscript bin/trimming_summary.R rawReads/ trimmedReads/ Report/figure/data/
```

**Output:** A csv table in the output directory.   
  

<br>   

### 10. Trimming QC plots   

**Make sure the arguments, specially *in_dir* and *out_dir* arguments, correspond to the trimmed reads corresponding folders**

**main script:** bin/fastqc_tables_and_plots.py   

**sub scripts:** bin/create_fastqcPlots_allSamples.R; bin/create_fastqcPlots_perSample.R; bin/create_fastqcTables.py

**Arguments main script:**     

--------------------------------------------------------------------------------------------------
-h, --help   show this help message and exit
-v, --version        show program's version number and exit
--in_dir IN_DIR       Path to folder containing fastq files. Default=rawReads/
--out_dir_report OUT_DIR_REPORT Path to out put folder. Default=Report/figure
--suffix_name SUFFIX_NAME Suffix to optionally put to the output name. Default=\'\'
--sample_names_file SAMPLE_NAMES_FILE Text file with sample names. Default=sample_names.txt
--plot_device PLOT_DEVICE Specify the format of the plot output. Default=png
--------------------------------------------------------------------------------------------------

**Command example:**  

```bash
$ python bin/fastqc_tables_and_plots.py --in_dir trimmedReads --out_dir_report Report/figure/trimmedQC/ --suffix_name _trimmed --sample_names_file sample_names.txt
```

<br>   

### 10 (Optional for Lexogen) Trim polyA sequences   

PolyA sequences may also affect the mapping of reads if the alignment tool used involves alignment of reads from end-to-end. The tool used for mapping in this analysis is STAR, as explained in the following section.

STAR does local alignment and clipping of 3’-end of reads in case that such an alignment gives a better score. This means that the majority of polyA sequences are clipped from reads during the alignment step. For this reason it is not strictly necessary to trim polyA sequences from the reads. 

To remove polyA sequences I have used prinseq (argument -trim_right 8, but may change). 


### 11 Mapping   

**main script:** mappingReads.py   

**software:** samtools, STAR

**Define the reference genome in the analysis_info.txt**   

** Define the number of cores to be used in analysis_info.txt, normally 2 works fine**   

**Comments:** a temp dir can be a local directory. Transfer of data between servers (i.e fs3 and cluster) is often very slow.


**Arguments main script:**   

--------------------------------------------------------------------------------------------------
-h, --help   show this help message and exit
-v, --version        show program's version number and exit
--in_dir IN_DIR       Path to folder containing fastq files. Default=trimmedReads/
--out_dir OUT_DIR     Path to out put folder. Default=alignedReads/
--temp_dir TEMP_DIR   Path to a temp dir folder. Default=none
--sample_names_file SAMPLE_NAMES_FILE. Text file with sample names to map. Default=sample_names_info.txt
--------------------------------------------------------------------------------------------------

**Command example:**   
```bash
$ python bin/mappingReads.py --temp_dir ~/temp_mapping
```

### 11 Mapping QC






