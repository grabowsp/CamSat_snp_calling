# Process for choosing and prepping libraries for SNP calling

## Choosing libraries to include for Pop Genomics SNP calling
### Overview
*  Downloaded .xml file from JGI containing info about all the available \
C.sativa files available from JGI
* Processed the .xml file in R to get library names of resequencing \
libraries and filter out names not to be included 
* Transferred list of libraries to NERSC
### Gettin info from JGI
* Location of files from the overall Camelina sativa project on the JGI \
Genome Portal
  * `https://genome.jgi.doe.gov/portal/pages/dynamicOrganismDownload.jsf?organism=GenResaOilTraits`
  * note: need to sign in to JGI Genome Portal to access the info
* Location of xml file in HA system
  * `/home/t4c1/WORK/grabowsk/data/Camelina_general/camelina_jgi_files.xml`
### Generating list of libraries to process using R
#### Overview
* Found all fastq.gz files withing the 'resequencing' portion of the XML file
* Filtered out following types of libraries:
  * Any libraries of Camelina alyssum - this is a sister species of C.sativa
  * Any 'progeny' libraries so as not to over-represent specific families \
during the SNP calling process
  * Libraries generated from bulking DNA from multiple samples, since the \
genotypes from these libraries won't fit the same criteria as the rest of \
the libraries
#### R script
* `/home/t4c1/WORK/grabowsk/data/Camelina_general/camelina_xml_manipulation.r`
#### List of libraries to be included for Resequencing SNP Discovery
* `/home/t4c1/WORK/grabowsk/data/Camelina_general/camsat_reseq_libs_for_SNPcalling_1.txt`
  * Total of 232 libraries
### Location of Library List at NERSC
* `/global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1`

## Prepare Directory for for libarary prepping
### Overview
* Directory on NERSC:
  * `/global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/`
* Use Chris's prepper program to prepare the libraries for SNP calling
  * Info about prepper found in shared Google Doc
    * `https://docs.google.com/document/d/1ISRXR1s6WHnNDBQSN0eTu1-11gajfdHRouUMX03Ev3Y/edit`
* Requires setting up many subdirectories
  * Ex: CONFIG, linkerSeq, contaminant, directories with each library code...
  * Info found in GoogleDoc
### Generate Subdirectories
* Generate necessary subdirectories and copy files to `linkerSeq` directory
```
bash
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
mkdir CONFIG
mkdir contaminant
mkdir linkerSeq
cp /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/linkerSeq/* ./linkerSeq
for i in `cat camsat_reseq_libs_for_SNPcalling_1.txt`; do mkdir ./$i; done
```

## Generate lib.config file
### Overview
* Requires using the jamo module/function to get information about the \
status and location of the libraries at JGI
  * I had to work with the NERSC Help Desk to get me access to the jamo \
module on Cori
* Used `jamo info LIB_CODE` to get information about the libraries
  * Any 'PURGED' libraries had to be 'fetched' using \
`jamo fetch library LIB_CODE`
  * from what I understand, this brings back the library from tape storage \
so it can be directly accessed during the prepping
* Use `jamo report short_form library LIB_CODE` to get info about the \
libraries that could be put into the `lib.config` file
  * Need to include a `grep UNPROCESSED` term so that info about all \
libraries tied to a `LIB_CODE` are included in the prepping
### Get status of libraries
```
bash
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in `cat camsat_reseq_libs_for_SNPcalling_1.txt`;
do jamo info library $i >> reseq_lib_info.txt; done
```
### Fetch and purged libraries
* Use `grep PURGED reseq_lib_info.txt` to get `LIB_CODE` of any libraries \
that were not in the main storage of JGI
* Use `jamo fetch library LIB_CODE` for those libraries
* ex:
```
for i in `grep PURGED reseq_lib_info.txt`;
do jamo fetch library $i; done
```
* Need to wait until `jamo info library LIB_CODE` shows a state of \
`BACKUP_COMPLETE` or `RESTORED` before going on to next step
### Generate lib.config file
```
bash
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in `cat camsat_reseq_libs_for_SNPcalling_1.txt`;
do jamo report short_form library $i | grep UNPROCESSED >> ./CONFIG/lib.config;
done
```

## Check that libraries pass QC
### Overview
* use jamo to see if any `LIB_CODE`s are linked to fastq files that failed \
the JGI QC test
* compare the Failed QC test files to those in lib.config
### Looked for Failed runs
* need to load the jamo module
```
module load jamo
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in `cat camsat_reseq_libs_for_SNPcalling_1.txt`;
do jamo report short_form library $i | grep 'passedQC=F' >> qc_F_report.txt;
echo $i; done
```
### Check if failed runs are included in lib.config
```
qc_report_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/qc_F_report.txt'
qc_report <- read.table(qc_report_file, header = F, sep = ' ',
  stringsAsFactors = F)
#
lib_config_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/CONFIG/lib.config'
lib_config <- read.table(lib_config_file, header = F, sep = '\t',
  stringsAsFactors = F)
#
f_info <- strsplit(qc_report[,4], split = '/')
f_files <- unlist(lapply(f_info, function(x) x[[length(x)]]))
#
f_in_lc_list <- sapply(f_files, function(x) grep(x, lib_config[,6], fixed = T))
f_in_lc_length <- unlist(lapply(f_in_lc_list, length))
f_in_lc <- which(f_in_lc_length > 0)
length(f_in_lc) > 0
# [1] FALSE
```
* no overlap between QC failure and files in lib.config

## Submit prepping jobs
### Overview
* Generated sub-lists of 20 `LIB_CODE`s because launching the prepper submits \
many jobs for each `LIB_CODE`, and Chris recommended prepping 20 libraries \
at a time to stay withing the rules of NERSC
* Submitted first prepper job on Denovo because was having trouble with Cori
  * I worked with the NERSC Help Desk people to get everything resolved \
on Cori - mainly it was getting added to the correct Unix groups
* Submitted second prepper job on Cori to test that it worked
* Then submitted jobs in groups
### Generate sub-lists
* Generate sub-lists of 20 `LIB_CODE` entries for submitting jobs
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
split -l 20 -d camsat_reseq_libs_for_SNPcalling_1.txt lib_small_list_
```
### Submit test job on Denovo
```
bash
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
cd ./GCNCU
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1 GCNCU
```
### Submit test job on Cori
```
bash
module load python
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
cd ./GGNUZ
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1 GGNUZ
```
### Submit jobs in groups on Cori
* for first `LIB_CODE` sublist, already prepper first 2, so only need to \
run on last 18
```
bash
module load python
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/ 
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in `tail -n 18 lib_small_list_00`;
do cd ./$i
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1 $i
cd ..
sleep 10s;
done
```
* for rest of sublists, use `cat` in the `for i in ...` portion of the loop
* `lib_small_list_01`
```
bash
module load python
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in `cat lib_small_list_01`;
do cd ./$i
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1 $i
cd ..
sleep 10s;
done
```
* `lib_small_list_02`, `lib_small_list_03`, `lib_small_list_04`, \
`lib_small_list_05`, `lib_small_list_06`
* for `lib_small_list_07` and `lib_small_list_08`; setting '-q 100' so can \
launch more jobs - fewer jobs run simultaneously per prep, but don't have to \
do quite as sessions submitting jobs
  * same for `lib_small_list_09`, `libs_small_list_10`, and `lib_small_list_11`
```
bash
module load python
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in `cat lib_small_list_11`;
do cd ./$i
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1 $i -q 100
cd ..
sleep 10s;
done
```

## Adjustments for `LIB_CODE`s with multiple fastq files
### Overview
* Several library codes were sequenced more than once so have more than one \
fastq file associated with the `LIB_CODE`
  * `CAWTG`, `CAWTH`, `CAWTN`, `CAWTO`
* For these cases, each fastq file should be assiciated with a slightly \
different `LIB_CODE`
  * ex: `LIB_CODE`; `LIB_CODE_1`, etc
* Need to change the lib.config file so that the raw fastq files are \
associated with the different `LIB_CODE` name variants
* Then rerun the prep
* For now, will do the re-runs on denovo since there's capacity
### Adjust names in lib.config
* `CAWTG`, `CAWTH`, `CAWTN`, `CAWTO`
* Changed second fastq `LIB_CODE` to `LIB_CODE_1`
* Also changed "PROCESSED" to "UNPROCESSED"; though not sure that's necessary
### Clear out previous prep output
* remove all the files from those 4 directories
### Make additional directories
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
mkdir CAWTG_1
mkdir CAWTH_1
mkdir CAWTN_1
mkdir CAWTO_1
```
### Submit preps on denovo
```
bash
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in CAWTG CAWTG_1 CAWTH CAWTH_1 CAWTN CAWTN_1 CAWTO CAWTO_1; 
do cd ./$i 
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1 $i
cd ..
sleep 10s;
done
```

## Check that preps finished properly
### Overview
* check that there is a `prepComplete` file in each `LIB_CODE` directory
* For any libraries that did not complete, figure out what the error was
  * grep for errors in slurm.out files
### Check that all libraries have `prepComplete` file
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in `cat camsat_reseq_libs_for_SNPcalling_1.txt`; 
do ls -oh ./$i/prepComplete; done
```
* `CAWTH` is the only one that failed
### Identify problem in CAWTH
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/CAWTH
for i in `ls slurm*out`; do grep 'PREP FAILED' $i; done
```
* output: `CAWTH_65 PREP FAILED - Read counts do not match`
* manually correct the pid for `CAWTH_65`
  * originally had pid 20642225
  * should be 20642243
### Submit combine job
```
sbatch CAWTHCombine.sh
```
### Check that all libraries are prepper properly
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in `ls -d */`; do ls -oh $i/prepComplete; done
```
* all `LIB_CODE` directories contain a 'prepComplete' file


## Archive Prepped Libraries
### Overview
* will use Chris's custom script to archive prepped libraries into jamo
* Example:
```
python /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/generalizedArchiver.py [CONFIG FILE]
```
  * use -h for help/to see options
  * use -e to see example of format for CONFIG FILE
### Generate archiving CONFIG FILE using R
* Load prepping lib.config file to get the `LIB_CODE`s
* Use that info to generate Location, Meta_data, etc
* Use the lib.config file to get the original file name
```
prep_config_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/CONFIG/lib.config'
prep_config <- read.table(prep_config_file, header = F, sep = '\t', 
  stringsAsFactors = F)

prep_data_dir <- '/global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/'
destination_dir <- 'SNPS/Camelina_sativa'

archive_list <- list()
for(pc in seq(nrow(prep_config))){
  tmp_list <- list()
  tmp_lib_name <- prep_config[pc, 1]
  tmp_list[['FILE']] <-paste(tmp_lib_name, '.prepped.fastq.bz2', sep = '')
  tmp_list[['LOCATION']] <- paste(prep_data_dir, tmp_lib_name, '/', 
    sep = '')
  tmp_list[['DESTINATION']] <- destination_dir
  tmp_list[['META_DATA']] <- paste(prep_data_dir, tmp_lib_name, '/', 
    tmp_lib_name, '.extractionStats', sep = '')
  tmp_orig_full <- unlist(strsplit(prep_config[pc, 6], split = '/'))
  tmp_list[['ORIGFILE']] <- tmp_orig_full[length(tmp_orig_full)]
  tmp_list[['FILETYPE']] <- 'fastq,fastq.gz'
  tmp_list[['STATE']] <- 'UNPROCESSED'
  #
  archive_list[[tmp_lib_name]] <- tmp_list
}

archive_df <- data.frame(
  matrix(unlist(archive_list), byrow = T, ncol = length(archive_list[[1]])),
  stringsAsFactors = F)

colnames(archive_df) <- names(archive_list[[1]])

archive_config_out <- paste(prep_data_dir, 'reseq_archive_config.txt', 
  sep = '')

write.table(archive_df, file = archive_config_out, quote = F, sep = '\t', 
  row.names = F, col.names = T)
```
### Run Archiving Script
```
bash
module load python
module load jamo
# source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
python /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/generalizedArchiver.py reseq_archive_config.txt

```


