[![Downloads](https://img.shields.io/github/downloads/Nextomics/NextPolish/total.svg)](https://github.com/Nextomics/NextPolish/releases/download/v1.0.2/NextPolish.tgz)
[![Release](https://img.shields.io/github/release/Nextomics/NextPolish.svg)](https://github.com/Nextomics/NextPolish/releases)
[![Last-commit](https://img.shields.io/github/last-commit/Nextomics/NextPolish.svg)](https://github.com/Nextomics/NextPolish/commits/master)
[![Issues](https://img.shields.io/github/issues/Nextomics/NextPolish.svg)](https://github.com/Nextomics/NextPolish/issues)

# NextPolish
NextPolish is used to fix base errors (SNP/Indel) in the genome generated by noisy long reads, it can be used with short read data only or long read data only or a combination of both. It contains two core modules and four steps (steps 3 & 4 are optional), and use a stepwise fashion to correct the error bases in reference genome. To correct the raw third-generation sequencing (TGS) long reads with approximately 15-10% sequencing errors, please use [NextDenovo](https://github.com/Nextomics/NextDenovo).

* **DOWNLOAD**  
click [here](https://github.com/Nextomics/NextPolish/releases/download/v1.0.2/NextPolish.tgz) or use the following command:  
`wget https://github.com/Nextomics/NextPolish/releases/download/v1.0.2/NextPolish.tgz`  

* **REQUIREMENT**
	* [Python 2.7](https://www.python.org/download/releases/2.7/)
	* [Shutil](https://docs.python.org/2/library/shutil.html)
	* [Signal](https://docs.python.org/2/library/signal.html)
	* [Drmaa](https://github.com/pygridtools/drmaa-python)

* **INSTALL**  
`cd NextPolish && make`

* **UNINSTALL**  
`cd NextPolish && make clean`

* **RUN**  
`nextPolish test_data/run.cfg`

>*If the raw genome generated without a consensus step, such as [miniasm](https://github.com/lh3/miniasm), please run the following command or [racon](https://github.com/isovic/racon) 2-3 rounds using long reads only before running [NextPolish](https://github.com/Nextomics/NextPolish), to avoid many short reads that cannot be mapped correctly due to the high error rate in the raw genome.*

```bash
    threads=20  
    genome=input.genome.fa
    lgsreads=input.lgs.reads.fq.gz
    bin/minimap2 -a -t ${threads} -x map-ont/map-pb ${genome} ${lgsreads}|bin/samtools view -F 0x4 -b - |bin/samtools sort - -m 2g -@ ${threads} -o genome.lgs.bam  
    bin/samtools index -@ ${threads} genome.lgs.bam  
    python lib/nextPolish.py -g ${genome} -t 5 --bam_lgs genome.lgs.bam -p ${threads} > genome.lgspolish.fa
```

* **INPUT** 

<pre>
	[General]                # global options
	job_type = sge           # [local, sge, pbs...]. (default: sge)
	job_prefix = nextPolish  # prefix tag for jobs. (default: nextPolish)
	task = best              # task need to run [all, default, best, 1, 2, 3, 4, 12, 123, 12123...], all=1234, default=12, best=12121212. (default: default)
	rewrite = no             # overwrite existed directory [yes, no]. (default: no)
	rerun = 3                # re-run unfinished jobs untill finished or reached ${rerun} loops, 0=no. (default: 3)
	parallel_jobs = 30       # number of tasks used to run in parallel. (default: 30)
	multithread_jobs = 20    # number of threads used to in a task. (default: 20)
	cluster_options = -l vf={vf} -q all.q -pe smp {cpu} -S {bash} -w n
	                         # a template to define the resource requirements for each job, which will pass to <a href="https://github.com/pygridtools/drmaa-python/wiki/FAQ">DRMAA</a> as the nativeSpecification field.
	genome = genome.fa       # genome file need to be polished. (<b>required</b>)
	genome_size = auto       # genome size, auto = calculate genome size using the input ${genome} file. (default: auto)
	workdir = 01_rundir      # work directory. (default: ./)
<!-- 	round_count = 1          # number of iterations to run NextPolish cyclically. (default: 1)
	round_mode = 2           # preset mode of iterations to run NextPolish cyclically, 1 = 1234[1234], 2 = 12[12]34, 3 = 123[123]4. (default: 2) -->
	polish_options = -p {multithread_jobs}
	                         # options used in polished step, see below.

	[sgs_option]             # options of short reads. (<b>required</b>)
	sgs_fofn = ./sgs.fofn    # input short reads file, one file one line, paired-end files should be interleaved.
	sgs_options = -max_depth 100 -bwa
	                         # -use_duplicate_reads, use duplicate pair-end reads in the analysis. (default: False)
	                         # -unpaired, unpaired input files. (default: False)
	                         # -max_depth, use up to ${max_depth} fold reads data to polish. (default: 100)
	                         # -bwa, use bwa to do mapping. 
	                         # -minimap2, use minimap2 to do mapping, which is much faster than bwa. (default: -minimap2)

	[lgs_option]             # options of long reads. (<b>optional</b>)
	lgs_fofn = ./lgs.fofn    # input long reads file, one file one line.             
	lgs_options = -min_read_len 1k -max_read_len 150k -max_depth 60
	                         # -min_read_len, filter reads with length shorter than ${min_read_len}. (default: 1k)
	                         # -max_read_len, filter reads with length longer than $ {max_read_len}, ultra-long reads usually contain lots of errors, and the mapping step requires significantly more memory and time. (default: 150k)
	                         # -max_depth, use up to ${max_depth} fold reads data to polish. (default: 60)
	lgs_minimap2_options = -x map-pb -t {multithread_jobs}
	                         # minimap2 options, used to set PacBio/Nanopore read overlap. (<b>required</b>)
	
	[polish_options]         # options used in polished step.
	-debug                   # output details of polished bases to stderr. (default: False)

	algorithm arguments:
	-count_read_ins_sgs      # read ${count_read_ins_sgs} reads to estimate the insert size of paired-end reads. (default: 10000)
	-min_map_quality         # skip the mapped read with mapping quality < ${min_map_quality}. (default: 0)
	-max_ins_len_sgs         # skip the paired-end read with insert size > ${max_ins_len_sgs}. (default: 10000)
	-max_ins_fold_sgs        # skip the paired-end read with insert size > ${max_ins_fold_sgs} * estimated_average_insert_size. (default: 5)
	-max_clip_ratio_sgs      # skip the mapped read with clipped length > ${max_clip_ratio_sgs} * full_length, used for bam_sgs. (default: 0.15)
	-max_clip_ratio_lgs      # skip the mapped read with clipped length > ${max_clip_ratio_lgs} * full_length, used for bam_lgs. (default: 0.4)
	-trim_len_edge           # trimed length at the two edges of a alignment. (default: 2)
	-ext_len_edge            # extened length at the two edges of a low quality region. (default: 2)

	score_chain:
	-indel_balance_factor_sgs 
	                         # a factor to control the ratio between indels, larger factor will produced more deletions, and vice versa. (default: 0.5)
	-min_count_ratio_skip    # skip a site if the fraction of the most genotype > ${min_count_ratio_skip}. (default: 0.8)

	kmer_count:
	-min_len_ldr             # minimum length requirement of a low depth region, which will be further processed using bam_lgs. (default: 3)
	-max_len_kmer            # maximum length requirement of a polished kmer, longer kmers will be splited. (default: 50)
	-min_len_inter_kmer      # minimum interval length between two adjacent kmers, shorter interval length will be merged. (default: 5)
	-max_count_kmer          # read up to this count of observed kmers for a polished kmer. (default: 50)

	snp_phase:
	-ploidy                  # set the ploidy of the sample of this genome. (default: 2)
	-max_variant_count_lgs   # exclude long reads with more than ${max_variant_count_lgs} variable sites, it is approximately equivalent to total error bases in the long read. (default: 150k)
	-indel_balance_factor_lgs 
	                         # a factor to control the ratio between indels, larger factor will produced more deletions, and vice versa. (default: 0.33)
	-min_depth_snp           # recall snps using bam_lgs if the total depth of this site in bam_sgs < ${min_depth_snp}. (default: 3)
	-min_count_snp           # recall snps using bam_lgs if the count of this snp in bam_sgs < ${min_count_snp}. (default: 5)
	-min_count_snp_link      # find a snp linkage using bam_lgs if the count of this linkage in bam_sgs < ${min_count_snp_link}. (default: 5)
	-max_indel_factor_lgs    # recall indels with bam_sgs if the count of the second most genotype > ${max_indel_factor_lgs} * the count of the most genotype when the most genotype is different with ref in bam_lgs. (default: 0.21)
	-max_snp_factor_lgs      # recall snps with bam_lgs if the count of the second most genotype > ${max_snp_factor_lgs} * the count of the most genotype when the most genotype is different with ref. (default: 0.53)
	-min_snp_factor_sgs      # skip a snp if the count of the second most genotype < ${min_snp_factor_sgs} * the count of the most genotype. (default: 0.34)
</pre>

* **OUTPUT**    
genome.nextpolish.partnnn.fasta with fasta format, the fasta header includes primary seqID, length.

* **HELP**   
Please raise an issue at the issue page.

* **CONTACT INFORMATION**    
For additional help, please send an email to huj_at_grandomics_dot_com.

* **COPYRIGHT**    
NextPolish is freely available for academic use and other non-commercial use. 

* **PLEASE STAR AND THANKS**    

* **FAQ**  
	1. Which job scheduling systems are supported by NextPolish?  
	NextPolish use [DRMAA](https://en.wikipedia.org/wiki/DRMAA) to submit, control, and monitor jobs, so in theory, support all DRMAA-compliant system, such as LOCAL, SGE, PBS, SLURM.
	2. How to continue running unfinished tasks?  
	No need to make any changes, simply run the same command again.
	3. Is it necessary to run steps 3 and 4?  
	In most cases, you can only run steps 1 and 2. For high-heterozygosity, high-repetitive, or polyploid genomes, you can try steps 3 and 4 using LGS data.
	4. How many iterations to run NextPolish cyclically to get the best result?  
	Our test shown that run NextPolish with 4 iterations, and almost all the bases with effectively covered by SGS data can be corrected. Please set task = best to get the best result. Set task = 12121212 means NextPolish will cyclically run steps 1 and 2 with 4 iterations.
	5. What is the difference between bwa or minimap2 to do SGS data mapping?  
	Our test shown Minimap2 is about 3 times faster than bwa, but the accuracy of polished genomes using minimap2 or bwa is tricky, depending on the error rate of genomes and SGS data, see [here](https://lh3.github.io/2018/04/02/minimap2-and-the-future-of-bwa) for more details.
	6. How to specify the queue name/cpu/memory/bash to submit jobs?  
	Please use cluster_options, NextPolish will replace {vf}, {cpu}, {bash} with specific values needed for each jobs.
	7. RuntimeError: Could not find drmaa library.  Please specify its full path using the environment variable DRMAA_LIBRARY_PATH.   
	Please setup the environment variable: DRMAA_LIBRARY_PATH, see [here](https://github.com/pygridtools/drmaa-python) for more details.
	8. ERROR: drmaa.errors.DeniedByDrmException: code 17: error: no suitable queues.    
	This is usually caused by a wrong setting of cluster_options, please check cluster_options first. If you use SGE, you also can add '-w n' to cluster_options, it will switch off validation for invalid resource requests. Please add a similar option for other job scheduling systems. 
	9. OSError: /path/lib64/libc.so.6: version `GLIBC_2.14' not found (required by /path/NextPolish/lib/calgs.so).  
	Please download [this version](https://github.com/Nextomics/NextPolish/releases/download/v1.0.2/NextPolish1.0.0-CentOS6.9.tgz) and try again.
