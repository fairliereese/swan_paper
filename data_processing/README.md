# Data processing and TALON
This page contains all the commands used to generate the files used as input to Swan for the manuscript.

## Download mapped PacBio RNA-seq data for each replicate
```bash
# hepg2 rep 1
wget https://www.encodeproject.org/files/ENCFF575ABA/@@download/ENCFF575ABA.bam
mv ENCFF575ABA.bam hepg2_1.bam

# hepg2 rep 2
wget wget https://www.encodeproject.org/files/ENCFF463JKW/@@download/ENCFF463JKW.bam
mv ENCFF463JKW.bam hepg2_2.bam

# hffc6 rep 1
wget https://www.encodeproject.org/files/ENCFF677TYB/@@download/ENCFF677TYB.bam
mv ENCFF677TYB.bam hffc6_1.bam 

# hffc6 rep 2
wget https://www.encodeproject.org/files/ENCFF584UQW/@@download/ENCFF584UQW.bam
mv ENCFF584UQW.bam hffc6_2.bam

# hffc6 rep 3
wget https://www.encodeproject.org/files/ENCFF964PGS/@@download/ENCFF964PGS.bam
mv ENCFF964PGS.bam hffc6_3.bam
```

## Convert each bam to sam 
```bash
module load samtools
for file in *.bam;
do
	sam_file=`basename $file .bam`
	sam_file=${sam_file}.sam
	samtools view -h $file > $sam_file
done
```

## Label reads with TALON for detection of internal priming
```bash
FASTA=~/mortazavi_lab/ref/hg38/hg38.fa
mkdir labelled
for file in *.sam;
do
	base=`basename $file .sam`
	talon_label_reads \
		--f $file \
		--g $FASTA \
		--t 8 \
		--tmpDir ${base}_temp \
		--deleteTmp \
		--o labelled/${base}
done
```

## Initialize TALON database with GENCODE v29 annotation
```bash 
GTF=~/mortazavi_lab/ref/gencode.v29/gencode.v29.annotation.gtf
talon_initialize_database \
	--f $GTF \
	--g hg38 \
	--a gencode_v29 \
	--l 0 \
	--idprefix TALON \
	--5p 500 \
	--3p 300 \
	--o talon
```

## Create config file to run TALON 
```bash
plat="Sequel"
touch config.csv
for file in labelled/*.sam;
do
	base=`basename $file .sam`
	base="${base::${#base}-8}"
	sample="${base::${#base}-2}"
	printf "${base},${sample},${plat},${file}\n" >> config.csv
done
```

## Run TALON
```bash
talon \
	--f config.csv \
	--db talon.db \
	--t 16 \
	--build hg38 \
	--cov 0.9 \
	--identity 0.8 \
	--o talon_swan
```

## Filter annotated transcripts for internal priming and reproducibility for each cell line
```bash
talon_filter_transcripts \
	--db talon.db \
	-a gencode_v29 \
	--datasets hepg2_1,hepg2_2 \
	--maxFracA 0.5 \
	--minCount 5 \
	--minDatasets 2 \
	--o hepg2_whitelist.csv

talon_filter_transcripts \
	--db talon.db \
	-a gencode_v29 \
	--datasets hffc6_1,hffc6_2,hffc6_3 \
	--maxFracA 0.5 \
	--minCount 5 \
	--minDatasets 2 \
	--o hffc6_whitelist.csv
```

## Create a GTF of annotated transcripts for each replicate
```bash
printf "hepg2_1" > dataset_hepg2_1
talon_create_GTF \
	--db talon.db \
	-b hg38 \
	-a gencode_v29 \
	--whitelist hepg2_whitelist.csv \
	--datasets dataset_hepg2_1 \
	--o hepg2_1
printf "hepg2_2" > dataset_hepg2_2
talon_create_GTF \
	--db talon.db \
	-b hg38 \
	-a gencode_v29 \
	--whitelist hepg2_whitelist.csv \
	--datasets dataset_hepg2_2 \
	--o hepg2_2

printf "hffc6_1" > dataset_hffc6_1
talon_create_GTF \
	--db talon.db \
	-b hg38 \
	-a gencode_v29 \
	--whitelist hffc6_whitelist.csv \
	--datasets dataset_hffc6_1 \
	--o hffc6_1
printf "hffc6_2" > dataset_hffc6_2
talon_create_GTF \
	--db talon.db \
	-b hg38 \
	-a gencode_v29 \
	--whitelist hffc6_whitelist.csv \
	--datasets dataset_hffc6_2 \
	--o hffc6_2
printf "hffc6_3" > dataset_hffc6_3
talon_create_GTF \
	--db talon.db \
	-b hg38 \
	-a gencode_v29 \
	--whitelist hffc6_whitelist.csv \
	--datasets dataset_hffc6_3 \
	--o hffc6_3
```

## Obtain a filtered abundance file of each transcript that passed filtering in either of the cell lines
```bash
cat hepg2_whitelist.csv > all_whitelist.csv
cat hffc6_whitelist.csv >> all_whitelist.csv 
sort all_whitelist.csv | uniq > whitelist.csv
talon_abundance \
	--db talon.db \
	-a gencode_v29 \
	-b hg38 \
	--whitelist whitelist.csv \
	--o all
```