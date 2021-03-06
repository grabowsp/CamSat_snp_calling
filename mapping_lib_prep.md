# Process for choosing and prepping libraries for SNP calling

## Generate list of libraries to include
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
### Generate list of libraries using R
#### Overview
* Found all fastq.gz files withing the 'Raw Data' portion of the XML file \
that either have 'Mapping Population' or 'Parent' in their Seq Project Name
#### R script
* `/home/t4c1/WORK/grabowsk/data/Camelina_general/camsat_mapping_xml_manipulation.r`
#### Location of list on HA
* `/home/t4c1/WORK/grabowsk/data/Camelina_general/camsat_mapping_libs_for_SNPcalling_1.txt`
  * total of 265 libraries
### Location at NERSC
* `/global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/camsat_mapping_libs_for_SNPcalling_1.txt`

## Prepare Directory for the library prepping
### Overview
* Directory on NERSC:
  * `/global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/`
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
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1
mkdir CONFIG
mkdir contaminant
mkdir linkerSeq
cp /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/linkerSeq/* ./linkerSeq
for i in `cat camsat_mapping_libs_for_SNPcalling_1.txt`; do mkdir ./$i; done
```

## Generate lib.config file
### Overview
* Requires using the jamo module/function to get information about the \
status and location of the libraries at JGI
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
module load python
module load jamo
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1
for i in `cat camsat_mapping_libs_for_SNPcalling_1.txt`;
do jamo info library $i >> reseq_lib_info.txt; done
```
### Check that all libraries are on tape
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
module load python
module load jamo
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1
for i in `cat camsat_mapping_libs_for_SNPcalling_1.txt`;
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
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1
for i in `cat camsat_mapping_libs_for_SNPcalling_1.txt`; 
do jamo report short_form library $i | grep 'passedQC=F' >> qc_F_report.txt; 
echo $i; done
```
### Check if failed runs are included in lib.config
```
qc_report_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/qc_F_report.txt'
qc_report <- read.table(qc_report_file, header = F, sep = ' ', 
  stringsAsFactors = F)
#
lib_config_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/CONFIG/lib.config'
lib_config <- read.table(lib_config_file, header = F, sep = '\t', 
  stringsAsFactors = F)
#
f_info <- strsplit(qc_report[,5], split = '/')
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
* Generated sub-lists of 45 `LIB_CODE`s because launching the prepper submits \
many jobs for each `LIB_CODE`, and Chris recommended prepping 45 libraries \
at a time if use '-q 100' flag to stay withing the rules of NERSC
### Generate sub-lists
* Generate sub-lists of 45 `LIB_CODE` entries for submitting jobs
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1
split -l 45 -d camsat_mapping_libs_for_SNPcalling_1.txt lib_small_list_
```
### Submit jobs
* have submitted: `lib_small_list_00`, `lib_small_list_01`, \
`lib_small_list_02`, `lib_small_list_03`, `lib_small_list_04`, \
`lib_small_list_05`
```
bash
module load python
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1
for i in `cat lib_small_list_05`;
do cd ./$i
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1 $i -q 100
cd ..
sleep 10s;
done
```

## Check if libraries have more than one fastq file
```
lib_config_file <-'/global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/CONFIG/lib.config'
lib_config <- read.table(lib_config_file, header = F, sep = '\t', 
  stringsAsFactors = F)

sum(duplicated(lib_config[,1]))
# 0
```

## Checking that preps have finished
### Overview
* check that there is a `prepComplete` file for each library
* for those that didn't finish, check what happened
### Check for `prepComplete` file
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1
for i in `cat camsat_mapping_libs_for_SNPcalling_1.txt`; 
do ls -oh ./$i/prepComplete; done
```
* all finished for `lib_small_list_00`
* 3 did not finish in `lib_small_list_01`
  * GBYGB, GBYHA, GBYNN didn't finish..
* When running for full list, all have finished

### Troubleshooting
#### GBYGB
* `GBYGB_6` did not run/finish
* Need to: 
  * rerun `GBYGB_6.sh`
  * update `GBYGB.pids` with correct pid for GBYGB_6
  * run combiner job
##### Rerun `GBYGB_6`
```
cd grabowsp@cori11:/global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/GBYGB
sbatch GBYGB_6.sh
```
##### Update `GBYGB_6` pid
* update GBYGB.pids with new `GBYGB_6` pid (slurm output number)
* new pid is 21050998
##### Run combiner job
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/GBYGB
sbatch GBYGBCombine.sh
```
#### GBYHA
* looks like it may have run out of time
* will re-run the prep
  * need to delete the GBYHA.pids file
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/GBYHA
rm GBYHA.pids
module load python
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1 GBYHA -q 100
```
#### GBYNN
* looks like it may have run out of time
* will re-run the prep
  * need to delete the GBYNN.pids file
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/GBYNN
rm GBYNN.pids
module load python
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1 GBYNN -q 100
```

## Archive Prepped Libraries
### Overview
* Use Chris's script
* Need to generate a CONFIG file for the archiver
  * use the lib.config file to get important info
### Generate CONFIG file
* in R
```
prep_config_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/CONFIG/lib.config'
prep_config <- read.table(prep_config_file, header = F, sep = '\t',
  stringsAsFactors = F)

prep_data_dir <- '/global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/'
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

archive_config_out <- paste(prep_data_dir, 'mapping_archive_config.txt',
  sep = '')

write.table(archive_df, file = archive_config_out, quote = F, sep = '\t',
  row.names = F, col.names = T)
```
### Run Archiving Script
```
bash
module load python
module load jamo
cd /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1
python /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/generalizedArchiver.py mapping_archive_config.txt
```


