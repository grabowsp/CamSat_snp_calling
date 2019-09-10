# Using default SNP caller for remaining Camelina libraries

## Overview
* Generate subdirectories for SNP calling
* Generate list of LIB_CODEs
* Generate generic snps.config for multi-fasta libraries
* Copy and adjust multi-fasta snps.config files
* Make softlinks in Prep directories for multi-fasta libraries
* Generate generic snps.config for single-fasta libraries
* Copy and adjust single-fasta snsps.config files
* Make sublists of 15 LIB_CODEs
* Run snpCaller
* Check quality of snpCaller output
* Clean up directories

## Generate subdirectores for SNP calling
### Data directories
* Directory for SNP calling
  * `/global/projectb/scratch/grabowsp/Cs_remain_snp_calling`
* Directory from library prep
  * `/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call`
### Generate list of LIB_CODEs manually
```
cp /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/remain_master_libcodes.txt \
/global/projectb/scratch/grabowsp/Cs_remain_snp_calling/remain_libcodes.txt
```
* use vim to remove duplicate LIB_CODEs
### Generate subdirectories
```
cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling
for i in `cat remain_libcodes.txt`;
do mkdir $i; done
```

## Generate multi-fasta snps.config files
### Generate generic multi-fasta snps.config
```
cp /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/CAWTG/snps.config \
/global/projectb/scratch/grabowsp/Cs_remain_snp_calling/\
multi_fa_generic_snps.config
```
* adjusted the read directory using vim
### Make list of libcodes with more than 1 fasta
```
cp /global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/\
multi_fasta_names.txt \
/global/projectb/scratch/grabowsp/Cs_remain_snp_calling/multi_fasta_libs.txt
```
* take out the `_1` using vim
### Generate multi-fasta snps.config for each multi-fasta file
```
cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling
for LIB in `cat multi_fasta_libs.txt`;
do sed 's/LIB_CODE/'"$LIB"'/g' multi_fa_generic_snps.config \
> ./$LIB/snps.config;
done
```

## Generate soft links for additional prepped fastq files
```
for LIB in `cat /global/projectb/scratch/grabowsp/Cs_remain_snp_calling/\
multi_fasta_libs.txt`;
do ln -s \
/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/\
$LIB'_1/'$LIB'_1.prepped.fastq.bz2' \
/global/projectb/scratch/grabowsp/Csativa_reseq/remain_libs_call/\
$LIB'/'$LIB'_1.prepped.fastq.bz2';
done
```

## Generate single-fasta snps.config files
### Make list of single-fasta libs
* `/global/projectb/scratch/grabowsp/Cs_remain_snp_calling/\
single_fasta_libs.txt`
  * make using vim
### Make generic single-fasta snps.config
```
cp /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/GBSTA/snps.config \
/global/projectb/scratch/grabowsp/Cs_remain_snp_calling/\
single_fa_generic_snps.config
```
* adjust with vim
### Make snps.config
```
cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling
for LIB in `cat single_fasta_libs.txt`;
do sed 's/LIB_CODE/'"$LIB"'/g' single_fa_generic_snps.config \
> ./$LIB/snps.config;
done
```

## Generate sub-lists
```
cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling
split -l 15 -d remain_libcodes.txt remain_sub_
```

## Start SNPcaller
* ran: 00, 01
```
cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling
for LIB in `cat remain_sub_01`;
do cd ./$LIB
snpCaller_GP.py
sleep 1s
cd ..;
done
```

## QC
* check that final VCF was generated
* checked: 00
```
bash
cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling
for i in `cat remain_sub_01`;
do ls -oh ./$i'/'$i'.GATK.SNP.postFilter.vcf';
done
```

## Troubleshooting
### Overview
* usually, the issue is that the merging step times out, I think, so need to \
restart from that step
### samples that needed to be restarted
* 01
  * GBSYU, GBSYN
#### check if jobs are still running before restarting
```
cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling/
for LC in GCBWB GCBWZ GCBWA;
  do
  sacct | grep $LC | grep RUN;
  sacct | grep $LC | grep PEND;
  done
```
### Code for re-starting stalled pipeline
* change the library codes at beginning of loops
```
for LC in GBSYU GBSYN;
  do cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling/$LC;
  for x in $(ls -l *.sorting.sh.stderr | awk '{if($5>122)print $9}' | rev \
| cut -f2- -d"." | rev);
    do echo $x; sbatch $x; sleep 1s;
    done;
  done
```
* next step:
```
for LC in GBSYU GBSYN;
  do cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling/$LC;
  snpCaller_GP.py -s merge_bam_files;
  sleep 1s;
  done
```

## Clean up libraries after finishing with SNP calling
* ran for: 00, 01
```
bash
cd /global/projectb/scratch/grabowsp/Cs_remain_snp_calling/
for LIB in `cat remain_sub_01`;
do cd ./$LIB
rm -f *.sh.* *fastq.gz *.sorting.sh *.merging.sh
rm -f *.RG.ba* *.deDup.ba* *.reAligned.ba*
rm -f *.GRP_?.?.bam *.GRP_??.?.bam *.GRP_??.??.bam *.GRP_?.??.bam *.GRP_?.???.bam *.GRP_??.???.bam
rm -f *.GRP_?.?.sorted.bam *.GRP_??.?.sorted.bam *.GRP_??.??.sorted.bam *.GRP_?.??.sorted.bam *.GRP_?.???.sorted.bam *.GRP_??.???.sorted.bam
cd ..;
done
```

