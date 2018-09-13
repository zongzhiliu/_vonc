# extract the variant info from FoundationMedicine data from dbGAP, calculate the allele frequencys.
* indication level: #patients with the variant/#total patients
* gene level: #patients with the gene mutated/#total patients
* variant in gene: #patients with the variant/#patients with the gene mutated

## Browse and Download data from [GDC here](https://gdc.cancer.gov/about-gdc/contributed-genomic-data-cancer-research/foundation-medicine/foundation-medicine)
### download detailed clinical data from [Clinical and Biospecimen]()
### download vcf files (annotated somatic mutations) from [Short variants]
* download via GDC data transfer tool (*.vcf.gz, *.vep.vcf.gz, n=18004+18004)


## liftover from hg38 to hg19
```
#install requirements
conda install -c bioconda ucsc-liftover ucsc-twobittofa picard
#download chain file
cd ~/data
wget http://hgdownload.cse.ucsc.edu/goldenpath/hg38/liftOver/hg38ToHg19.over.chain.gz
#download genome file
wget http://hgdownload.soe.ucsc.edu/goldenPath/hg19/bigZips/hg19.2bit
twoBitToFa hg19.2bit hg19.fa

#lift over with picard
picard -Xmx10g CreateSequenceDictionary R=hg19.fa O=hg19.dict
    #? liftOver failed because of pos+1 and mismatching reference, should be a bug of picard?
    #: NO, it is a mistake of FM

# lift over with liftover: bed [0,1), vcf,igv [1,1], 
cat test1.vcf | sed '/^#/d' | awk '{ print $1, $2-1, $2, $8}' > test1.tmp
liftOver test1.tmp ~/data/hg38ToHg19.over.chain.gz test1.o test1.e
```
Turn out that the FM liftover to hg38 make a position+1 shift, so that a liftover from the downloaded vcf to hg19 give errors of mismatching reference. Decided to simply replace the Chrom and POS with the originalContig and INFO.originalStart.

## test run snpEff
```
conda install -c bioconda snpeff
snpEff databases | grep -i human
snpEff -Xmx4g -v GRCh37.p13.RefSeq test1.hg19.vcf > test1.hg19.ann.vcf
```
SnpEff on 18K file will be very time consuming, thinking of generate a combined vcf file.

## fix and merge the vcf files
- Fix: by replace CHOME with INFO.orginalChrom, and POS with INFO.originalPos, then replace the head with hg19
- Merge: merge the 18 vcf files by add a column sampleId with the Values defined by FORMAT
- Implemented by matthew

## run snpEff
```
#test run
vepVcf='/scratch/Foundation_Medicine/VCFS/combined_variants.vep.vcf' 
head $vepVcf -n 15 > test2.vep.vcf
#snpEff -Xmx4g -v GRCh37.p13.RefSeq test2.vep.vcf > test2.vep.eff.vcf
#manual check INFO.ANN: 
    #? it is consistence with vep in all cases for genename, c. p., effect, etc
#run
vepEffVcf='/scratch/Foundation_Medicine/VCFS/combined_variants.vep.snpEff.vcf'
snpEff -Xmx10g -v GRCh37.p13.RefSeq $vepVcf > $vepEffVcf
```
## check the snpEff results
* snpEff_summary.html
    * warnings and errors
    * number of processed: yes
    * variant rate details: NM?  >> check the snpEff ref
    * number of variants by type
    * number of effects by impact: most moderate, then High, modifier (as expected, as they are prefiltered by FM)
    * Number of effects by functional class: Misssense, nonsense, silent?  >> check for inconsistence
    * Number of effects by type and region: Exon, upstream, intron, downstream, splicing, UTR, ... >> check the up and downstream
    * ins/del lengths
    * base changes for SNPs: G2A/C2T is highly enriched?  >> research
    * Ts/Tv ratio = 1.4
    * Amino acid changes: E2K, R2H enriched
* snpEff_genes.txt
    todo: summarise by gene name
* vep.eff.vcf

## error and warning messages 
    #> dig them out later
```
ERRORS: Some errors were detected
Error type      Number of errors
ERROR_CHROMOSOME_NOT_FOUND      16

WARNINGS: Some warning were detected
Warning type    Number of warnings
INFO_REALIGN_3_PRIME    6238
WARNING_TRANSCRIPT_INCOMPLETE   2197
WARNING_TRANSCRIPT_INCOMPLETE&INFO_REALIGN_3_PRIME      2
WARNING_TRANSCRIPT_MULTIPLE_STOP_CODONS 8950
WARNING_TRANSCRIPT_MULTIPLE_STOP_CODONS&INFO_REALIGN_3_PRIME    30
WARNING_TRANSCRIPT_NO_START_CODON       3052
WARNING_TRANSCRIPT_NO_STOP_CODON        186
```

## add the snpEff annotated vcf_merged to database
* run allele_normalization
* report the vcf headers: anno of the fields and INFO subfields
* implemented by matthew

## calculate allele freq
### test: do gene level first, no filtering, on the whole cohort
* clincal.csv: > sampleId, tumorType # later using disease type from MAF files (sampleSheet)
* vcfCombined:  Hugo_Symbol, sampleId
    * using python parser: 
    ```
    cat test2.vep.vcf | sed '/^#/d' | python ~/github/vonc/main.py 
    cat /scratch/Foundation_Medicine/VCFS/OLD_BACKUPS/combined_variants.vep.snpEff.vcf | sed '/^#/d' | python ~/github/vonc/main.py > variant_call.csv
    ```
```calcFreq.R```

### prepare the patient cancertype data
* download the 42 clinical files and merger them
```
python merge_clinical.py */*.tsv > clinical_merged.csv
```
* download the TCGA mapping from the FM paper supplemental table 3 and do the mapping
```R
tmp = read.csv('clinical.gdc_download_20180911_180824.513845/clinical_merged.csv')
clinical = tmp
res = sqldf('select maf_group, `diagnoses.primary_diagnosis`, count(*) as count_patient from tmp group by maf_group, `diagnoses.primary_diagnosis` order by maf_group, count_patient desc') #where count_patient != 0') #sort(table(diagnoses.primary_diagnosis))
write.csv(res, 'FM_to_TCGA.csv')
```
* manually mapping in excel, and then create a view with patient, tcga_type
```R
mapping = read.csv('FM_to_TCGA.manual_mapping.csv', row.names=1)
```
* upload to the database
```R
require(RMySQL)
require(configr)
mycnf = read.config('~/.my.cnf')$client
dbname='var_dbGAP_phs001179_v1_p1_20180823'
con = dbConnect(MySQL(), host=mycnf$host, port=integer(mycnf$port), user=mycnf$user, password=mycnf$password, dbname=dbname)
dbWriteTable(con, name='clinical_merged', value=clinical)
dbWriteTable(con, name='mapping_to_TCGA', value=mapping)
dbDisconnect(con)
```

## create views to calculate AIs
```sql
use var_dbGAP_phs001179_v1_p1_20180823;

create VIEW `v_clinical_tcga` AS (
    select `cases.submitter_id` AS `CaseID`,`maf_group`, `diagnoses.primary_diagnosis` AS `primary_diagnosis`, `TCGA_type` AS `cancer_type`
    from (`clinical_merged` `c` join `mapping_to_TCGA` m using (`diagnoses.primary_diagnosis`, `maf_group`)
    where (TCGA_type != '');

create VIEW `v_cancertype_count` AS (
    select cancer_type, count(*) AS total_patient 
    from `v_clinical_tcga` group by `cancer_type`);

create view `v_variant_call` AS (
select distinct replace(SUBMITTER_ID, '_sample','') AS `CaseID`, `symbol`,`HGVSp`
from `vep_snpEff_normed_rvs_v3` `v` join `FM_Annotations_v3` f using (vkey)
where annotation_position_index = 0
); 

#performance tweak
create table tv_variant_call as (select * from v_variant_call);
create index idx_tv_variant_call_CaseID on tv_variant_call (CaseID);
#change data type of caseID, cancer_type, HGVS to VARCHAR
create table tv_clinical_tcga as ( select * from v_clinical_tcga);
alter table tv_clinical_tcga add primary key (CaseID);

# gene freq
create table tv_cancertype_gene_count as (
select cancer_type, symbol, count(distinct `CaseID`) as count_patient
    from tv_clinical_tcga join tv_variant_call using (`CaseID`)
    group by cancer_type, `symbol`
    );

create view v_cancertype_gene_freq as (
select cancer_type, symbol, 100.*count_patient/total_patient as freq
from tv_cancertype_gene_count join v_cancertype_total using (cancer_type)
order by cancer_type, freq desc
);

# indication freq
create table tv_cancertype_indication_count as (
select cancer_type, symbol, HGVSp, count(distinct `CaseID`) as count_patient
    from tv_clinical_tcga join tv_variant_call using (`CaseID`)
    group by cancer_type, `symbol`, HGVSp
    );

create view v_cancertype_indication_freq as (
select cancer_type, symbol, HGVSp, 100.*count_patient/total_patient as freq
from tv_cancertype_indication_count join v_cancertype_total using (cancer_type)
order by cancer_type, freq desc
);
# performance tweaks
# add index (cancer_type, symbol) in both tables

#variant in gene freq
create view v_cancertype_variant_in_gene as (
select cancer_type, symbol, HGVSp, 100.*i.count_patient/g.count_patient as freq
from tv_cancertype_indication_count i join tv_cancertype_gene_count g using (cancer_type, symbol)
order by cancer_type, freq desc);


## global allele freq
### inication freq
create view v_cohort_indication_count as (
select symbol, HGVSp, sum(count_patient) as count_patient
from tv_cancertype_indication_count
group by symbol, HGVSp
);
# performance tweak
create index idx_cancertype_indication_count on tv_cancertype_indication_count (symbol, HGVSp(100));
create table tv_cohort_indication_count as (select * from v_cohort_indication_count);
create index idx_cohort_indication_count on tv_cohort_indication_count (symbol, HGVSp(100));

drop view v_cohort_indication_freq;
create view v_cohort_indication_freq as (
select symbol, HGVSp, 100.*count_patient/(select sum(total_patient) from v_cancertype_total) as freq
from tv_cohort_indication_count
order by freq desc
);

### gene freq
drop view v_cohort_gene_count;
create view v_cohort_gene_count as (
    select symbol, sum(count_patient) as count_patient
    from tv_cancertype_gene_count group by symbol
);

drop view v_cohort_gene_freq;
create view v_cohort_gene_freq as (
select symbol, 100. * count_patient/(select sum(total_patient) from v_cancertype_total) as freq
from v_cohort_gene_count
order by freq desc
);

### variant in gene
drop view v_cohort_variant_in_gene;
create view v_cohort_variant_in_gene as (
select symbol, HGVSp, 100. * i.count_patient/g.count_patient as freq
from tv_cohort_indication_count i join v_cohort_gene_count  g  using (symbol)
order by freq desc
);

```

## test solution with sqldf
```R
tv_variant_call = dbReadTable(con, 'tv_variant_call')
tv_clinical_tcga = dbReadTable(con, 'tv_clinical_tcga')
v_cancertype_total = dbReadTable(con, 'v_cancertype_total')
options(RMySQL.dbname='user_zach_liu')
tv_cancertype_gene_count = sqldf('select cancer_type, symbol, count(distinct CaseID) count_patient
    from tv_clinical_tcga join tv_variant_call using (`CaseID`)
    group by cancer_type, `symbol`')
```
