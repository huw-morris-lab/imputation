# imputation
Imputation steps using the Michigan or Sanger imputation server.

Date: April 2020

Last updated: 22/04/2020

Authors: Manuela Tan

## General description and purpose

This covers:
1. Pre-imputation preparation
2. Uploading your data onto the imputation server (Michigan or Sanger server)
3. Post-imputation filtering

Make sure you have done your QC steps first (see the snp-array-QC project).

However, there is debate about how much SNP filtering to do before you before imputation. I have done all the standard QC as in the snp-array-QC repository, but there are papers which suggest that including more SNPs is better for imputation quality: https://www.ncbi.nlm.nih.gov/pubmed/25112433


# 1. Michigan

Read the guidance at https://imputationserver.sph.umich.edu/

Download the latest version of Will Rayner's checking tool: https://www.well.ox.ac.uk/~wrayner/tools/

```
#Check what the latest version is
wget http://www.well.ox.ac.uk/~wrayner/tools/HRC-1000G-check-bim.v4.2.13.zip

#Extract files from zip
unzip HRC-1000G-check-bim-v4.2.13.zip
```

Get allele frequencies (from your final QC-ed binary files)
```
plink --bfile NeuroChip_v1.1_run3-10.QSBB_PD.geno_0.95.maf_0.01.sampleqc.sample_0.98.het_2SD.updatedsex.sexpass.hwe.IBD_0.1.PCA_keep \
--freq \
--out NeuroChip_v1.1_run3-10.QSBB_PD.geno_0.95.maf_0.01.sampleqc.sample_0.98.het_2SD.updatedsex.sexpass.hwe.IBD_0.1.PCA_keep.freq
```

Run the Will Rayne checking tool perl script. 

The sites file is available in /data/kronos/NGS_Reference/HRC.r1-1.GRCh37.wgs.mac5.sites.tab

```
perl HRC-1000G-check-bim.pl \
-b NeuroChip_v1.1_run3-10.QSBB_PD.geno_0.95.maf_0.01.sampleqc.sample_0.98.het_2SD.updatedsex.sexpass.hwe.IBD_0.1.PCA_keep.bim \
-f NeuroChip_v1.1_run3-10.QSBB_PD.geno_0.95.maf_0.01.sampleqc.sample_0.98.het_2SD.updatedsex.sexpass.hwe.IBD_0.1.PCA_keep.freq.frq \
-r /data/kronos/NGS_Reference/HRC.r1-1.GRCh37.wgs.mac5.sites.tab \
-h
```

Check your output.

This will make a new shell script called Run-plink.sh. Usually you just run this with sh Run-plink.sh

However at the moment this is running into errors with the plink --real-ref-allele flag. Try with different version of plink - this may be because the version on kronos is older.


Download plink v1.9 into your directory and then edit the shell script to use this version instead. Wherever the script refers to plink, direct it to the newer version you have just downloaded. E.g.  /data/kronos/mtan/software/plink_1-9/plink

I saved the shell script as Run-plink-edited.sh

```
sh Run-plink-edited.sh
```

This Will Rayne script splits your data into separate chromosomes and makes the VCF files.

Now create sorted vcf.gz file (loop over all chromosomes). I renamed the final files with a shorter name
```
sh
for chr in {1..23}
do
	FILENAME$chr.vcf | bgzip -c > preimpute_FILENAME$chr.vcf.gz
done
```

Upload to the Michigan imputation server https://imputationserver.sph.umich.edu/

These are the settings I have used but you can also select other reference panels etc.

	Ref panel: HRC r1.1 2016
	rsq filter: OFF
	Phasing: Eagle v2.4
	Population: EUR
	Mode: Quality control and imputation


The TOPMed imputation server and reference panel has also recently been made available: https://imputation.biodatacatalyst.nhlbi.nih.gov/. I am not sure what population groups are - ALL vs. Mixed


	Ref panel: TOPMed
	rsq filter: OFF
	Phasing: Eagle v2.4
	Population: All
	Mode: Quality control and imputation
