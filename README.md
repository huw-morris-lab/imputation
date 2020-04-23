# imputation
Imputation steps using the Michigan or Sanger imputation server.

Date: April 2020

Last updated: 23/04/2020

Authors: Manuela Tan

## General description and purpose

This covers:
1. Pre-imputation preparation
2. Uploading your data onto the imputation server (Michigan server)
3. Post-imputation filtering

Make sure you have done your QC steps first (see the snp-array-QC project).

However, there is debate about how much SNP filtering to do before you before imputation. I have done all the standard QC as in the snp-array-QC repository, but there are papers which suggest that including more SNPs is better for imputation quality: https://www.ncbi.nlm.nih.gov/pubmed/25112433


# 1. Prepare data for imputation

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

# 2. Impute on the Michigan Server (or TOPMed)

Upload your vcf.gz files to the Michigan imputation server https://imputationserver.sph.umich.edu/. Further guidance at https://imputationserver.readthedocs.io/en/latest/.

These are the settings I have used but you can also select other reference panels etc.

	Ref panel: HRC r1.1 2016
	rsq filter: OFF
	Phasing: Eagle v2.4
	Population: EUR
	Mode: Quality control and imputation


The TOPMed imputation server and reference panel has also recently been made available: https://imputation.biodatacatalyst.nhlbi.nih.gov/.

I am not sure what population groups are - ALL vs. Mixed

	Ref panel: TOPMed
	rsq filter: OFF
	Phasing: Eagle v2.4
	Population: All
	Mode: Quality control and imputation

The TOPMed reference panel is in hg38 but the server automatically lifts over your data. https://imputationserver.readthedocs.io/en/latest/getting-started/#build

# 3. Post-imputation filtering

Read about bcftools http://samtools.github.io/bcftools/bcftools.html.

Files are downloaded in zip format and you need to use the password you were emailed to unzip these.
```
sh
for chr in {1..23}
do
	unzip -P "yourpassword" chr_$chr.zip
done
```

This will unzip your zip folders and leave you with .dose.vcf.gz and .info.gz for each chromosome.

Now we use bcftools to combine all the separate chromosome files and filter by imputation quality. Adjust your cutoff R2 as needed - 0.8 is quite a high threshold, GWAS studies usually use 0.3.

```
#concat combines all the chromosomes into a single file
#view filters by info score
#norm normalises indels. Split multiallelic sites into biallelic records. SNPs and indels merged into a single record
#create final gzipped VCF file and annotate. Remove original SNP ID and assign new SNP ID as chrom:position:ref:alt

bcftools concat chr1.dose.vcf.gz chr2.dose.vcf.gz chr3.dose.vcf.gz chr4.dose.vcf.gz chr5.dose.vcf.gz chr6.dose.vcf.gz chr7.dose.vcf.gz chr8.dose.vcf.gz chr9.dose.vcf.gz chr10.dose.vcf.gz chr11.dose.vcf.gz chr12.dose.vcf.gz chr13.dose.vcf.gz chr14.dose.vcf.gz chr15.dose.vcf.gz chr16.dose.vcf.gz chr17.dose.vcf.gz chr18.dose.vcf.gz chr19.dose.vcf.gz chr20.dose.vcf.gz chr21.dose.vcf.gz chr22.dose.vcf.gz -Ou | 
bcftools view -Ou -i 'R2>0.3' |
bcftools norm -Ou -m -any |
bcftools norm -Ou -f  /data/kronos/NGS_Reference/fasta/human_g1k_v37.fasta |
bcftools annotate -Oz -x ID -I +'%CHROM:%POS:%REF:%ALT' -o new_allchromosomes.converted.R2_0.3.vcf.gz
```

This takes a long time to run, recommend saving this as a .sh script and then running with nohup. This just means that you can leave it and if you disconnect from the server your job will continue. Any output is written in nohup.out.
```
nohup sh script.sh &
```
Or use the HPC on kronos.


The next step is to use plink to convert VCF format into binary files. Optional lines have been commented out. --double-id means that both family and within-family IDs to be set to the sample ID
```
plink --vcf new_allchromosomes.converted.R2_0.3.vcf.gz \
--double-id \
--allow-extra-chr 0 \
--maf 0.01 \
--make-bed \
--out new_allchromosomes.converted.R2_0.3.MAF_0.01

#You can also include these other flags - depends on what you want to do
--vcf-min-GP 0.9 \ # If you want to filter by minimum posterior probability
--vcf-idspace-to _ \ # If you have any spaces in your IDs, it converts to _ because plink does not allow spaces in IDs
```


