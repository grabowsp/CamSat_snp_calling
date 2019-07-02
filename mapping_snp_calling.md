# Process for using defaul SNP Caller in Camelina RIL libraries

## Overview
* Generate subdirectories for SNP calling
* Generate list of `LIB_CODE`s
* Generate generic snps.config
* Copy and adjust snps.config for each directory
* Make sublists of 20 `LIB_CODE`s
* Run snpCaller
* Check quality of snpCaller output
* Clean up unneeded temporary files generated by snpCaller
### Notes
* Removed GBYGU, GBYNY
  * failed twice during snp calling

## Generate directories for SNP calling
### Overview
* use original file for choosing `LIB_CODE`s to generate directories
### Code
```
cd /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling
for i in `cat /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/camsat_mapping_libs_for_SNPcalling_1.txt`; do mkdir $i; done
```

## Generate list of `LIB_CODE`s
* copy list from prepping directory
```
cp /global/projectb/scratch/grabowsp/Csativa_reseq/camsat_map_call_1/camsat_mapping_libs_for_SNPcalling_1.txt /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/
```

## Generate generic snps.config
* copy from reseq/diversity directory
```
cp /global/projectb/scratch/grabowsp/Cs_reseq_snp_calling/snps.config_1file /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/
```
* adjust the 'READS' path so that it directs to correct prepped libraries

## Copy and adjust generic snps.config to each sub-directory
```
bash
cd /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling
for LIB in `cat camsat_mapping_libs_for_SNPcalling_1.txt`;
do
sed 's/TEST/'"$LIB"'/g' /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/snps.config_1file > /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/$LIB'/snps.config';
done
```

## Generate sub-lists
```
cd /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling
split -l 20 -d camsat_mapping_libs_for_SNPcalling_1.txt map_sub_
```

## Start SNPcaller
* ran: `map_sub_00`, `map_sub_01`, `map_sub_02`, `map_sub_03`, `map_sub_04` \
`map_sub_05`, `map_sub_06`, `map_sub_07`, `map_sub_08`, `map_sub_09`
```
cd /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/
for LIB in `cat map_sub_09`;
do cd ./$LIB
snpCaller_GP.py
sleep 1s
cd ..;
done
```

## QC
* check that final VCF was generated
* checked: 00, 01, 02, 03, 04, 05, 06, 07
```
bash
cd /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/
for i in `cat map_sub_08`;
do ls -oh ./$i'/'$i'.GATK.SNP.postFilter.vcf';
done
```

## Troubleshooting
### Overview
* usually, the issue is that the merging step times out, I think, so need to \
restart from that step
### samples that needed to be restarted
* `map_sub_00`
  * GBYAN, GBYAS, GBYAH, GBYAG, GBSYY
* `map_sub_01`
  * GBYAX, GBYBA, GBYBS
* `map_sub_02`
  * GBYCG, GBYCP
* `map_sub_03`
  * GBYGU - failed twice - will delete from overall list
* `map_sub_04`
  * GBYNY - failed twice - will delete from overall list
  * GBYOC, GBYNW, GBYNB, GBYNG, GBYNS, GBYOA
* `map_sub_05`
  * GCBHG, GCBHW, GCBHO, GCBHA, GCBHH - first
  * GCBGW, GCBHZ, GCBHC - second
* `map_sub_06`
  * GCBOT, GCBOB, GCBNT, GCBOP
  * GCBON
  * GCBOU
* `map_sub_07`
  * GCBPO, GCBPP
* `map_sub_08`
  * GCBST, GCBSP, GCBSY, GCBTN
  * GCBTC
#### check if jobs are still running before restarting
```
cd /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/
for LC in GCBTC GCBST GCBSP GCBSY GCBTN;
  do
  sacct | grep $LC | grep RUN;
  sacct | grep $LC | grep PEND; 
  done
```
### Code for re-starting stalled pipeline
* change the library codes at beginning of loops
```
for LC in GCBST GCBSP GCBSY GCBTN;
  do cd /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/$LC;
  for x in $(ls -l *.sorting.sh.stderr | awk '{if($5>122)print $9}' | rev \
| cut -f2- -d"." | rev);
    do echo $x; sbatch $x; sleep 1s;
    done;
  done
```
* next step:
```
for LC in GCBST GCBSP GCBSY GCBTN;
  do cd /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/$LC;
  snpCaller_GP.py -s merge_bam_files;
  sleep 1s;
  done
```

## Clean up libraries after finishing with SNP calling
* ran for: 00, 01, 02, 03, 04, 05, 06, 07
```
bash
cd /global/projectb/scratch/grabowsp/Cs_RIL_snp_calling/
for LIB in `cat map_sub_07`;
do cd ./$LIB
rm -f *.sh.* *fastq.gz *.sorting.sh *.merging.sh
rm -f *.RG.ba* *.deDup.ba* *.reAligned.ba*
rm -f *.GRP_?.?.bam *.GRP_??.?.bam *.GRP_??.??.bam *.GRP_?.??.bam *.GRP_?.???.bam *.GRP_??.???.bam
rm -f *.GRP_?.?.sorted.bam *.GRP_??.?.sorted.bam *.GRP_??.??.sorted.bam *.GRP_?.??.sorted.bam *.GRP_?.???.sorted.bam *.GRP_??.???.sorted.bam
cd ..;
done
```

