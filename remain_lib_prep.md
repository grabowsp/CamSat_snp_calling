# Process for prepping the remaining Camelina sativa libraries for SNP calling

## Choosing libraries to prep
### Overview
* Already prepped libraries for resequencing and mapping groups
  * Wanted to prep the remaining libraries for if/when we need them
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
* Found all fastq.gz files that were not part of the reseq or mapping lib sets
#### R script
* `/home/t4c1/WORK/grabowsk/data/Camelina_general/camelina_remaining_xml_manipulation.r`
#### List of libraries to be included for Resequencing SNP Discovery
* `/home/t4c1/WORK/grabowsk/data/Camelina_general/camsat_remaining_libs_for_SNPcalling_1.txt`
  * Total of 48 libraries
### Location of Library List at NERSC
* `/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call`

## Prepare Directory for for libarary prepping
### Overview
* Directory on NERSC:
  * `/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/`
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
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/
mkdir CONFIG
mkdir contaminant
mkdir linkerSeq
cp /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/linkerSeq/* ./linkerSeq
for i in `cat camsat_remaining_libs_for_SNPcalling_1.txt`; do mkdir ./$i; done
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
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/
for i in `cat camsat_remaining_libs_for_SNPcalling_1.txt`;
do jamo info library $i >> remain_lib_info.txt; done
```
### Fetch any purged libraries
* Use `grep PURGED remain_lib_info.txt` to get `LIB_CODE` of any libraries \
that were not in the main storage of JGI
* Use `jamo fetch library LIB_CODE` for those libraries
* ex:
```
for i in `grep PURGED remain_lib_info.txt`;
do jamo fetch library $i; done
```
* Need to wait until `jamo info library LIB_CODE` shows a state of \
`BACKUP_COMPLETE` or `RESTORED` before going on to next step
### Generate lib.config file
```
bash
module load python
module load jamo
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call
for i in `cat camsat_remaining_libs_for_SNPcalling_1.txt`;
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
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call
for i in `cat camsat_remaining_libs_for_SNPcalling_1.txt`;
do jamo report short_form library $i | grep 'passedQC=F' >> qc_F_report.txt;
echo $i; done
```
### Check if failed runs are included in lib.config
```
qc_report_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/qc_F_report.txt'
qc_report <- read.table(qc_report_file, header = F, sep = '@',
  stringsAsFactors = F)
#
lib_config_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/CONFIG/lib.config'
lib_config <- read.table(lib_config_file, header = F, sep = '\t',
  stringsAsFactors = F)
#
f_info <- strsplit(qc_report[,1], split = '/')
f_files <- unlist(lapply(f_info, function(x) x[[length(x)]]))
#
f_in_lc_list <- sapply(f_files, function(x) grep(x, lib_config[,6], fixed = T))
f_in_lc_length <- unlist(lapply(f_in_lc_list, length))
f_in_lc <- which(f_in_lc_length > 0)
length(f_in_lc) > 0
# [1] FALSE
```
* no overlap between QC failure and files in lib.config

## Check if libraries have more than one fastq file
* there are some `LIB_CODE`s with more than one fastq, so includes commands \
to alter the lib.config file and make list of new subdirectories to make
```
lib_config_file <-'/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/CONFIG/lib.config'
lib_config <- read.table(lib_config_file, header = F, sep = '\t',
  stringsAsFactors = F)

sum(duplicated(lib_config[,1]))
# 13

rep_tab <- table(lib_config[,1])

new_dir_vec <- c()

for(i in c(2:max(rep_tab))){
  tmp_lib_names <- names(which(rep_tab == i))
  for(tln in tmp_lib_names){
    tmp_inds <- which(lib_config[,1] == tln)
    tmp_dirs <- paste(
      lib_config[tmp_inds[2:length(tmp_inds)],1], seq(length(tmp_inds)-1), 
      sep = '_')
    lib_config[tmp_inds[2:length(tmp_inds)],1] <- tmp_dirs
    new_dir_vec <- c(new_dir_vec, tmp_dirs)
  }
}

write.table(lib_config, lib_config_file, quote = F, sep = '\t', row.names = F, 
  col.names = F)

multi_fasta_file <- paste(gsub('CONFIG/lib.config', '', lib_config_file, 
  fixed = T), 'multi_fasta_names.txt', sep = '')

write.table(new_dir_vec, multi_fasta_file, quote = F, sep = '\t', 
  row.names = F, col.names = F)
```
### Generate subdirectories for multi-fasta `LIB_CODE`s
```
bash
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call
for i in `cat multi_fasta_names.txt`;
do mkdir $i; done
```

## Submit prepping jobs
### Overview
* Create master list of subdirectories that includes original `LIB_CODE`s and \
altered `LIB_CODE`s from those with more than one fastq
* Generated sub-lists of 45 `LIB_CODE`s because launching the prepper submits \
many jobs for each `LIB_CODE`, and Chris recommended prepping 45 libraries \
at a time if use '-q 100' flag to stay withing the rules of NERSC
### Generate master list of subdirectories
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call
cat camsat_remaining_libs_for_SNPcalling_1.txt multi_fasta_names.txt > \
remain_master_libcodes.txt
```
### Generate sub-lists
* Generate sub-lists of 45 `LIB_CODE` entries for submitting jobs
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call
split -l 45 -d remain_master_libcodes.txt lib_small_list_
```
### Submit jobs
* have submitted: `lib_small_list_00`; `lib_small_list_00` 
```
bash
module load python
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call
for i in `cat lib_small_list_01`;
do cd ./$i
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call $i -q 100
cd ..
sleep 10s;
done
```

## Checking that preps have finished
### Overview
* check that there is a `prepComplete` file for each library
* for those that didn't finish, check what happened
### Check for `prepComplete` file
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call
for i in `cat remain_master_libcodes.txt`;
do ls -oh ./$i/prepComplete; done
```
* 4 did not finish
  * CPSZG; CPSZH; CPSZB; CPSZN
  * These are smRNA libraries and shouldn't be prepped here

## Troubleshooting
### Remove smRNA `LIB_CODE`s
* CPSZG, CPSZH, CPSZB, CPSZC, and CPSZN are smRNA libraries
* Need to:
  * Remove them from the list of libraries
  * Remove them their subdirectories
#### Remove smRNA codes from list of librareis
* `camsat_remaining_libs_for_SNPcalling_1.txt`
* `remain_master_libcodes.txt`
* `./CONFIG/lib.config`
* done manually with vim
#### Remove subdirectories
### Remove RNASEQ `LIB_CODE`s
#### Get info from lib.config
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/CONFIG
grep RNASEQ lib.config >> ../rnaseq_config_info.txt
```
#### Make list of `LIB_CODE`s and remove them from necessary files
* in R
```
rnaseq_info_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/rnaseq_config_info.txt'
rnaseq_info <- read.table(rnaseq_info_file, header = F, sep = '\t', 
  stringsAsFactors = F)

orig_libs_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/camsat_remaining_libs_for_SNPcalling_1.txt'
orig_libs <- read.table(orig_libs_file, header = F, stringsAsFactors = F)

master_libs_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/remain_master_libcodes.txt'
master_libs <- read.table(master_libs_file, header = F, stringsAsFactors = F)

config_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/CONFIG/lib.config'
config_info <- read.table(config_file, header = F, sep = '\t', 
  stringsAsFactors = F)

o_inds <- c()
m_inds <- c()
c_inds <- c()
for(rl in seq(nrow(rnaseq_info))){
  tmp_o_ind <- grep(rnaseq_info[rl, 1], orig_libs[,1])
  tmp_m_ind <- grep(rnaseq_info[rl, 1], master_libs[,1])
  tmp_c_ind <- grep(rnaseq_info[rl, 1], config_info[,1])
#
  o_inds <- c(o_inds, tmp_o_ind)
  m_inds <- c(m_inds, tmp_m_ind)
  c_inds <- c(c_inds, tmp_c_ind)
}

new_orig_libs <- orig_libs[-o_inds, ]
new_master_libs <- master_libs[-m_inds, ]
new_config_info <- config_info[-c_inds, ]

rnaseq_lib_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/rnaseq_libcodes.txt'
write.table(rnaseq_info[,1], file = rnaseq_lib_file, quote = F, sep = '\t',
  row.names = F, col.names = F)

write.table(new_orig_libs, file = orig_libs_file, quote = F, sep = '\t',
  row.names = F, col.names = F)
write.table(new_master_libs, file = master_libs_file, quote = F, sep = '\t',
  row.names = F, col.names = F)
write.table(new_config_info, file = config_file, quote = F, sep = '\t',
  row.names = F, col.names = F)
```
#### Remove directories
```
for i in `cat rnaseq_libcodes.txt`;
do rm -r ./$i; done
```

## Archive Prepped Libraries
### Overview
* Use Chris's script
* Need to generate a CONFIG file for the archiver
  * use the lib.config file to get important info
### Generate CONFIG file
* in R
```
prep_config_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/CONFIG/lib.config'
prep_config <- read.table(prep_config_file, header = F, sep = '\t',
  stringsAsFactors = F)

prep_data_dir <- '/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/'
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

archive_config_out <- paste(prep_data_dir, 'remain_archive_config.txt',
  sep = '')

write.table(archive_df, file = archive_config_out, quote = F, sep = '\t',
  row.names = F, col.names = T)
```
### Run Archiving Script
```
bash
module load python
module load jamo
cd /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call
python /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/generalizedArchiver.py remain_archive_config.txt
```

