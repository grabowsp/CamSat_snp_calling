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
```
bash
module load python
source activate /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/ANACONDA_ENVS/PREP_ENV/
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for i in `cat lib_small_list_06`;
do cd ./$i
python3 /global/dna/projectdirs/plant/geneAtlas/HAGSC_TOOLS/PREP_TESTING/splittingOPP.py /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1 $i
cd ..
sleep 10s;
done
```

