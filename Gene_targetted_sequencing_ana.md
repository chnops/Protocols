Gene-Targetted Sequence Analysis
====================

This protocol is specifically modified to use with Pat Schloss method of gene targetting sequencing results from which 3 fastq.gz files were yielded:
+ XXXX_I1_XXX.fastq.gz (index file that stores barcodes)
+ XXXX_R1_XXX.fastq.gz (forward paired-end sequence, no tag nor linker/primer combo)
+ XXXX_R2_XXX.fastq.gz (reverse paired-end sequence, no tag nor linker/primer combo)

*General Procedures:
1. Use RDP's pandaseq (RDP_Assembler) to construct good quality full length sequences (assem.fastq).   
2. Use RDPTools/SeqFilters to check barcodes quality and split them into different sample directories (ONLY barcodes need to be reverse complimented, sequences are in the correct orientation).   
3. Bin assembled sequences into different sample files.   
4. Check for chimeras.  
5. Use RDPTools/Classifiers to pick taxonomy, use RDPToools/AlignmentTools to align sequences and then cluster using RDPTools/mcCluster.   
6. R for analysis

1. RDP_assembler  
    1. First run with minimal constrants:
        ```
        ~/RDP_Assembler/pandaseq/pandaseq -N -o 10 -e 25 -F -d rbfkms -f /PATH/TO/Undetermined_S0_L001_R1_001.fastq.gz -r /PATH/TO/Undetermined_S0_L001_R2_001.fastq.gz 1> IGSB140210-1_assembled.fastq 2> IGSB140210-1_assembled_stats.txt
        ```

        You should look at the stat output. Pay close attention to sequences that are good in quality (Q > 27), large overlap (over 100) and short in assembled length (between 100 and 200). They are real sequence but not bacterial (e.g., some are parasidic worm genomes).  
    
    ```
    ~/RDP_Assembler/pandaseq/pandaseq -N -o 40 -e 25 -F -d rbfkms -l 220 -L 280 -f Undetermined_S0_L001_R1_001.fastq.gz -r Undetermined_S0_L001_R2_001.fastq.gz 1> IGSB140210-1_assembled_o40.fastq 2> IGSB140210-1_assembled_stats_o40.txt
    ```
    
    1. make sure the number of good assembled sequence in assembled.fastq is the same as the OK number in stats.txt file   
        ```
        grep -c "@M0" IGSB140210-1_assembled_o40.fastq  
        ```
2. RDPTools: SeqFilters   
    This step trims off the tag and linker sequences. Final files are splited into folders named by individual samples.    
    Need:   
    1. map file from Argonne
        ```
        #SampleID       BarcodeSequence LinkerPrimerSequence    Description
        DC1     TCCCTTGTCTCC    CCGGACTACHVGGGTWTCTAAT  DC1
        DC2     ACGAGACTGATT    CCGGACTACHVGGGTWTCTAAT  DC2
        DC3     GCTGTACGGATT    CCGGACTACHVGGGTWTCTAAT  DC3
        DC4     ATCACCAGGTGT    CCGGACTACHVGGGTWTCTAAT  DC4
        DC5     TGGTCAACGATA    CCGGACTACHVGGGTWTCTAAT  DC5
        ```

        Tags example: `TCCCTTGTCTCC`   

        Linker: `CC`
 
        Forward primer: 515F `GTGCCAGCMGCCGCGGTAA`
        
        Reverse primer: 806R `GGACTACHVGGGTWTCTAAT`

        **Note:**  
        The sequences (R1.fastq and R2.fastq) from ANL does not contain barcodes or primers! The tag information are stored in the index file (I1.fastq).   

    2. SeqFilters need tag file to be like this:   
        ```
        TCCCTTGTCTCC    DC1
        ACGAGACTGATT    DC2
        GCTGTACGGATT    DC3
        ATCACCAGGTGT    DC4
        TGGTCAACGATA    DC5
        ```

        1. You can parse the Argonne file into above format:
            ```
            python ~/Documents/Fan/Smita/micro_code_23/MiSeq_rdptool_map_parser.py ANL_MAPPING_FILE.txt > TAG_FILE.tag
            ```

        2. ANL sequences were sequenced from reverse primer end. The tag sequences need to be reverse compilmented (even though SeqFilter should automatically reverse compilment sequence if 16S genes are analyzed): 
            ```
            python revcomp.py 16S_tag.txt > 16S_tag_rev.txt
            ```
        
        3. Then run SeqFilters to parse the I1.fastq to bin seq id into individual sample directories.       
            ```
            java -jar $SeqFilters --seq-file original/Undetermined_S0_L001_I1_001.fastq.gz --tag-file 16S_tag_rev.txt --outdir initial_process_Index_rev
            ```

            Note: to make sure tags were binning as expected, the quality of the tag could be set as 0 to begin with `--min-qual 0`   

Fixing index id:
```
python ~/Documents/Fan/code/fix_index_fastqgz_names.py ~/Documents/ElizabethData/COBS_ITS/uploads/Undetermined_S0_L001_I1_001.fastq.gz ITS_I1_fixed_id.fastq
```

Subset index file:
```
$ python ~/Documents/Fan/code/subset_I_for_pandaseq.py ITS_assembled_o80.fastq.gz ITS_I1_fixed_id.fastq ITS_I1_fixed_assem_subset.fastq
```


Qiime: demultiplexing
```
$ split_libraries_fastq.py -i ITS_assembled_o80.fastq.gz -b ITS_I1_fixed_assem_subset.fastq --rev_comp_mapping_barcodes -p 0 -o slout_q20 -m ~/Documents/ElizabethData/COBS_ITS/validate_mapping_file/hofmockel_its_mapping_corrected.txt --store_qual_scores
```