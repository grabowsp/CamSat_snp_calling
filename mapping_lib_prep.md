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

# NEXT STEP IS TO PREPARE lib.config FILE, BUT I WANT TO WAIT UNTIL CLOSER
#  TO WHEN I WILL BE ABLE TO PREP THE FILES


