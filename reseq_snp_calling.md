# Notes about process of calling SNPs for Camelina resequencing libraries

## Overview


## Generate directories for SNP calling
### Overview
* use original file for choosing `LIB_CODE`s to generate directories
### Code
```
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling
for i in `cat /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/camsat_reseq_libs_for_SNPcalling_1.txt`; do mkdir $i; done
```

## SNP Calling for `LIB_CODE`s with multiple fastq files
### Overview
* Ran test with Jerry for CAWTG, so have properly setup snps.config file from \
that that can use as template for the other 3 `LIB_CODE`s that have 2 fastq \
files
* Step 1: copy and adjust the snps.config file using `sed`
* Step 2: generate soft-links for the additional prepped fastqs in the prep \
directory entitled `LIB_CODE`
  * For example, make soft linke for `LIB_CODE_1.prepped.fastq.bz2` in the \
`LIB_CODE` directory
* Step 3: Start `snpCaller_GP.py`
### Generate snps.config files for other 3 `LIB_CODE`s
```
for LIB in CAWTH CAWTN CAWTO;
do sed 's/CAWTG/'"$LIB"'/g' /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/CAWTG/snps.config \
> /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/$LIB/snps.config; done
```
### Generate soft links for additional prepped fastq files
* already did this for CAWTG when doing test with Jerry
```
cd /global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1
for LIB in CAWTH CAWTN CAWTO;
do ln -s \
/global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/$LIB'_1/'$LIB'_1.prepped.fastq.bz2' \
/global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/$LIB'/'$LIB'_1.prepped.fastq.bz2'; done
```
### Start snpCaller
```
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/
for LIB in CAWTH CAWTN CAWTO;
do cd ./$LIB
snpCaller_GP.py
sleep 10s
cd ..;
done
```
### Clean up directory after snpCalling
* commands from Jerry
* testing for CAWTG:
```
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/CAWTG
rm -f *.sh.* *fastq.gz *.sorting.sh *.merging.sh
rm -f *.RG.ba* *.deDup.ba* *.reAligned.ba*
rm -f *.GRP_?.?.bam *.GRP_??.?.bam *.GRP_??.??.bam *.GRP_?.??.bam *.GRP_?.???.bam *.GRP_??.???.bam
rm -f *.GRP_?.?.sorted.bam *.GRP_??.?.sorted.bam *.GRP_??.??.sorted.bam *.GRP_?.??.sorted.bam *.GRP_?.???.sorted.bam *.GRP_??.???.sorted.bam
```
* For other 3 `LIB_CODE`s:
```
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/
for LIB in CAWTH CAWTN CAWTO;
do cd ./$LIB
rm -f *.sh.* *fastq.gz *.sorting.sh *.merging.sh
rm -f *.RG.ba* *.deDup.ba* *.reAligned.ba*
rm -f *.GRP_?.?.bam *.GRP_??.?.bam *.GRP_??.??.bam *.GRP_?.??.bam *.GRP_?.???.bam *.GRP_??.???.bam
rm -f *.GRP_?.?.sorted.bam *.GRP_??.?.sorted.bam *.GRP_??.??.sorted.bam *.GRP_?.??.sorted.bam *.GRP_?.???.sorted.bam *.GRP_??.???.sorted.bam
cd ..;
done
```

## SNP Calling for `LIB_CODE`s with 1 single prepped fastq file
* Steps:
  * Generate list of `LIB_CODE`s
  * Generate generic snps.config
  * Copy and adjust snps.config to each directory
  * Make sublists of 20 `LIB_CODE`s
  * Start snp caller on one sublist at a time
### Generate list of `LIB_CODE`s
* will use the `LIB_CODE` file from prepping and take out the `LIB_CODE`s \
with 2+ fastq files
* In R:
```
lc_file <- '/global/projectb/scratch/grabowsp/Csativa_reseq/snp_call_1/camsat_reseq_libs_for_SNPcalling_1.txt'
lcs <- read.table(lc_file, header = F, sep = '\t', stringsAsFactors = F)
multi_lc <- c('CAWTG', 'CAWTH', 'CAWTN', 'CAWTO')
sing_lc <- setdiff(lcs[,1], multi_lc)

sing_lc_file <- '/global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/single_fastq_libCodes.txt'
write.table(sing_lc, file = sing_lc_file, quote = F, sep = '\t', 
  row.names = F, col.names = F)
```
### Generate generic single-fastq snps.config file
* copy snps.config from CAWTG and make adjustments to use as template for \
single fastq `LIB_CODE`s
### Copy and adjust generic snps.config to each sub-directory
```
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/
for LIB in `cat single_fastq_libCodes.txt`;
do
sed 's/TEST/'"$LIB"'/g' /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/snps.config_1file > /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/$LIB'/snps.config';
done
```
### Generate sub-lists
```
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/
split -l 20 -d single_fastq_libCodes.txt sing_lc_sub_
```
### Start SNPcaller
* ran: `sing_lc_sub_00`; `sing_lc_sub_01`; `sing_lc_sub_02`; `sing_lc_sub_03`;\
`sing_lc_sub_04`
```
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/
for LIB in `cat sing_lc_sub_04`;
do cd ./$LIB
snpCaller_GP.py
sleep 1s
cd ..;
done
```

## QC
### Overview
* check that final .bam files were generated
### Check that final bam and VCF was generated
* `sing_lc_sub_00`; `sing_lc_sub_01`; `sing_lc_sub_02`; `sing_lc_sub_03`
```
bash
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling
for i in `cat sing_lc_sub_03`;
do echo $i 
ls -oh ./$i'/'$i'.gatk.bam';
ls -oh ./$i'/'$i'.GATK.SNP.postFilter.vcf';
done
```

## Troubleshooting
### .bam sorting for GRP_20 did not finish
#### Background
* 7 libraries failed at the same spot: the sorting of the final .bam files \
for GRP_20
* Noticed because got "FAILED" message from SLURM in my email
#### Test Solution for GCNPC
* Jerry recommended trying the following
```
bash
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/GCNPC
for x in $(ls -l *.sorting.sh.stderr | awk '{if($5>0)print $9}' | rev \
| cut -f2- -d"." | rev);
do echo $x; sbatch $x; done
```
* once that is done, try this:
```
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/GCNPC
snpCaller_GP.py -s merge_bam_files
```
* Worked!
#### Run for remaining 6 libraries
```
for LC in GBSYY GBSXH GCNPU GCNPO GCNPB GCNPS;
  do cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/$LC;
  for x in $(ls -l *.sorting.sh.stderr | awk '{if($5>0)print $9}' | rev \
| cut -f2- -d"." | rev);
    do echo $x; sbatch $x; sleep 1s;
    done;
  done
```
* Next step:
```
for LC in GBSYY GBSXH GCNPU GCNPO GCNPB GCNPS;
  do cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/$LC;
  snpCaller_GP.py -s merge_bam_files;
  done
```
#### GCNPB didn't finish making vcf files
* trying this...
```
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/GCNPB
snpCaller_GP.py -s GC_Depth_plots
```
### Issue with `sing_lc_sub_02` libraries
* 6 libraries had issues:
  * GBPTZ; GBSZG; GBPTX; GGNUU; GGNWW; GBPUN
#### Try restarting the sorting part
```
bash
for LC in GBPTZ GBSZG GBPTX GGNUU GGNWW GBPUN;
  do cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/$LC;
  for x in $(ls -l *.sorting.sh.stderr | awk '{if($5>0)print $9}' | rev \
| cut -f2- -d"." | rev);
    do echo $x; sbatch $x; sleep 1s;
    done;
  done
```
* Next step:
```
for LC in GBPTZ GBSZG GBPTX GGNUU GGNWW GBPUN;
  do cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/$LC;
  snpCaller_GP.py -s merge_bam_files;
  done
```
### Issues with `sing_lc_sub_03` libraries
* Several libraries did not finish:
  * GBPSY, GBPXY, GBPUC, GGNWU, GBPZX, GBPWC
* Based on the SLURM error outputs, it looks like the failures occured \
at different points:
  * GBPWC looks to have failed during the merging (like most of the previous \
failures)
  * The others failed during the GC vs depth step, but looking at their \
sorting.sh.stderr outputs, it looks like they failed at same point, so will \
try re-starting them all at that same spot
#### Try restarting the sorting part
* note: it looks like any .sorting.sh.stderr file larger than 122 is a \
failed run, so will change the `awk` part of the command to reflect that
```
bash
for LC in GBPSY GBPXY GBPUC GGNWU GBPZX GBPWC;
  do cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/$LC;
  for x in $(ls -l *.sorting.sh.stderr | awk '{if($5>122)print $9}' | rev \
| cut -f2- -d"." | rev);
    do echo $x; sbatch $x; sleep 1s;
    done;
  done
```
* Next step:
```
for LC in GBPSY GBPXY GBPUC GGNWU GBPZX GBPWC;
  do cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/$LC;
  snpCaller_GP.py -s merge_bam_files;
  done
```



## Clean up libraries after finishing with SNP calling
* done for `sing_lc_sub_00`, `sing_lc_sub_01`, `sing_lc_sub_02`, \
`sing_lc_sub_02`
```
bash
cd /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/
for LIB in `cat sing_lc_sub_03`;
do cd ./$LIB
rm -f *.sh.* *fastq.gz *.sorting.sh *.merging.sh
rm -f *.RG.ba* *.deDup.ba* *.reAligned.ba*
rm -f *.GRP_?.?.bam *.GRP_??.?.bam *.GRP_??.??.bam *.GRP_?.??.bam *.GRP_?.???.bam *.GRP_??.???.bam
rm -f *.GRP_?.?.sorted.bam *.GRP_??.?.sorted.bam *.GRP_??.??.sorted.bam *.GRP_?.??.sorted.bam *.GRP_?.???.sorted.bam *.GRP_??.???.sorted.bam
cd ..;
done
```


