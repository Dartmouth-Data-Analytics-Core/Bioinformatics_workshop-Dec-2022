# Working with NGS data Part I

### Learning objectives: 
- Understand the FASTQ file format and the formatting sequence information it stores
- Learn how to perform basic operations on FASTQ files in the command-line 


```bash
#log on to a compute node if not already on one:
srun --nodes=1 --ntasks-per-node=1 --mem-per-cpu=4GB --cpus-per-task=1 --time=08:00:00 --partition=preempt1 --account=DAC --pty /bin/bash
source ~/.bash_profile

# activate the conda environment
conda activate bioinfo

#create a variable for the source directory 
SOURCE="/dartfs-hpc/scratch/fund_of_bioinfo"
```


## Raw NGS data, FASTQ file format
---

FASTQ files are a workhorse file format of bioinformatics, and contain sequence reads generated in next-generation sequencing (NGS) experiments. We often refer to FASTQ files as the *'raw'* data for an NGS experiment, although these are technically the BCL image files captured by the sequencer and are used to synthesize the FASTQ files.

FASTQ files contain four lines per sequence record:

Four rows exist for each record in a FASTQ file:
- *Line 1:* Header line that stores information about the read (always starts with an `@`), such as the instrument ID, flowcell ID, lane on flowcell, file number, cluster coordinates, sample barcode, etc.
- *Line 2:* The sequence of bases called
- *Line 3:* Usually just a `+` and sometimes followed by the read information in line 1
- *Line 4:* Individual base qualities (must be same length as line 2)

Here is what the first record of an example FASTQ file looks like
```
@SRR1039508.1 HWI-ST177:290:C0TECACXX:1:1101:1225:2130 length=63
CATTGCTGATACCAANNNNNNNNGCATTCCTCAAGGTCTTCCTCCTTCCCTTACGGAATTACA
+
HJJJJJJJJJJJJJJ########00?GHIJJJJJJJIJJJJJJJJJJJJJJJJJHHHFFFFFD
```

Quality scores, also known as **Phred scores**, on line 4 represent the probability the associated base call is incorrect, which are defined by the below formula for current Illumina machines:
```
Q = base quality
P = probability of incorrect base call

Q = -10 x log10(P)
or
P = 10^-Q/10
```

Intuitively, this means that a base with a Phred score (Q-score) of `10` has a `1 in 10` chance of being an incorrectly called base, or *90%* chance of being the correct base. Likewise, a score of `20` has a `1 in 100` chance (99% accuracy), `30` a `1 in 1000` chance (99.9%) and `40` a `1 in 10,000` chance (99.99%).

However, the quality scores, 4th line, are clearly not probabilities in the FASTQ file. Instead, quality scores are encoded by a character that is associated with an *ASCII (American Standard Code for Information Interchange)* characters. ASCII codes provide a convenient way of representing a number with a character.

In FASTQ files, Q-score is linked to a specific ASCII character by **adding 33 to the Phred-score**, and matching the resulting number with its ASCII character according to the standard code. This ensures quality scores only take up 1 byte per value, reducing the file size. The full table used for ASCII character to Q-score conversion is available [here](https://support.illumina.com/help/BaseSpace_OLH_009008/Content/Source/Informatics/BS/QualityScoreEncoding_swBS.htm).

Consider the first base call in our sequence example above, the `C` has a quality score encoded by an `H`, which corresponds to a Q-score of 39 (this information is in the linked table), meaning this is a good quality base call.

Generally, you can see this would be a good quality read if not for the stretch of `#`s indicating a Q-score of 2. Looking at the FASTQ record, you can see these correspond to a string of `N` calls, which are bases that the sequencer was not able to make a base call for. Stretches of Ns' are generally not useful for your analysis.

**Paired-end reads:**  

If you sequenced paired-end reads, you will have two FASTQ files:  
- *..._R1.fastq* - contains the forward reads  
- *..._R2.fastq*- contains the reverse reads  

<p align="left">
<img src="../figures/seq-config.png" title="xxxx" alt="context"
	width="60%" height="60%" />
</p>

Many bioinformatics softwares will recognize that such files are paired-end, and the reads in the forward file correspond to the reads in the reverse file, although you often have to specify the names of both files to these tools.

It is critical that the R1 and R2 files have the **same number of records in both files**. If one has more records than the other, which can sometimes happen if there was an issue in the demultiplexing process, you will experience problems using these files as paired-end reads in downstream analyses.

## Working with FASTQ files at the command line
----

To demonstrate how FASTQ files can be explored from the command line environment, we will be using an example set of FASTQ files generated in an RNA-seq study of human airway cell line and their reaction to glucocorticoids (described in [Himes *et al*, 2014, *PloS One*](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0099625)).

Raw sequence data was obtained from the [Sequence Read Archive (SRA)](https://www.ncbi.nlm.nih.gov/sra) under project accession [SRP033351](https://www.ncbi.nlm.nih.gov/sra?term=SRP033351), using the [SRA toolkit](https://github.com/ncbi/sra-tools) (SRA). The FASTQ files are stored in `/dartfs-hpc/scratch/fund_of_bioinfo/raw_full_fastq/`. To speed up computations in the workshop, these FASTQ files have been subset to only contain reads that align to chromosome 20.

```bash
# lets have a look at the project directory containing the reduced raw FASTQs
ls -lah $SOURCE/raw_fastq_files/

# lets have a look at the project directory containing the full raw FASTQs
ls -lah $SOURCE/raw_full_fastq/
```

Since these are paired-end reads each sample has a file for read 1 (SRRXXX_1) and a file for read 2 (SRRXXX_2). All of the files are `gzipped` in order to reduce the disk space they require, which is important as you can see that the full files are all **1GB** or more (you need a lot of space to process RNA-seq, or other-NGS data).

Given the size of these files, if everyone were to copy them to their home directory, this would take up a very large amount of disk space. Instead you will create a *symbolic link* or *symlink* to the data in the scratch drive with the command `ln -s`. This command is similar to `cp` in that it accepts a source file and destination path as the arguments.

```bash
# move into your fundamentals_of_bioinformatics directory
# check that it worked by running the pwd command.  You should be in your own directory on /dartfs-hpc/scratch.
cd $FOB

# lets keep our data organized and make a folder for these raw fastq files
mkdir raw
cd raw

# Create a symlink to the data directory in the scratch drive
ln -s $SOURCE/raw_fastq_files/*fastq.gz ./

# Check that your command worked
ls -lah
```
Any modifications made to the original files in `/dartfs-hpc/scratch/fund_of_bioinfo/raw_fastq_files/` will also be seen in the symlink files. Moving the original files or deleting the original files will cause the symlinks to malfunction.

Remember, because your symlinks are pointing to something in the scratch directory these files are slated to be deleted in 45 days, at which point your symlinks will still exist but no longer function properly.

### Basic operations

Generally you won't need to go looking within an individual FASTQ file, but for our purposes it is useful to explore them at the command line to help better understand their contents. Being able to work with FASTQ files at the command line can also be a valuable skill for troubleshooting problems that come up in your analyses.

Due to gzip compression of FASTQ files we have to unzip when we want to work with them. We can do this with the `zcat` command and a pipe (|). `zcat` works similar to `cat` but operates on zipped files, FASTQ files are very large and so we will use `head` to limit the output to the first ten lines.

Use `zcat` and `head` to have a look at the first few records in our FASTQ file.
```bash
# unzip and view first few lines of FASTQ file
zcat SRR1039508_1.chr20.fastq.gz | head
zcat SRR1039508_2.chr20.fastq.gz | head
```

How many records do we have in total? (don't forget to divide by 4..)
```bash
zcat SRR1039508_1.chr20.fastq.gz | wc -l
zcat SRR1039508_2.chr20.fastq.gz | wc -l
```
Remember, paired-end reads should have the same number of records!

What if we want to count how many adapter sequences exist in the FASTQ file? 

To do this, we would need to print all the sequence lines of each FASTQ entry, then using the pipe we can search the sequences for the adapter sequence. 

To print all the sequence lines (2nd line) of each FASTQ entry, we can use a command called `sed`, short for *stream editor* which allows you to streamline edits to text that are redirected to the command. You can find a tutorial on using `sed` [here](https://www.digitalocean.com/community/tutorials/the-basics-of-using-the-sed-stream-editor-to-manipulate-text-in-linux). The `sed` command often accepts a 'script' as an argument indicating the edits that should be made, the script argument is indicated by the text between single quotes.

First we will use `sed` with the `'p'` argument in the script to indicate we want the output to be printed, and the `-n` option to tell `sed` we want to run the command in silent mode. We specify `'2~4p'` as we want `sed` to *print line 2, then skip forward 4*. We can then pipe these results to the `head` command, we can get the sequence line of the first 10 entries in the FASTQ file. 
```bash
zcat SRR1039508_1.chr20.fastq.gz | sed -n '2~4p' | head
```

Building on this approach, we can print the second line for the first 10,000 entries of the FASTQ file, and use the `grep` command to search for the adapter sequence in the output. We use the `-o` option for grep, to print only the portion of the line that matches the character string.
```bash

# Pipe the sequence line from the first 10000 FASTQ records to grep to search for our (pretend) adapter sequence
zcat SRR1039508_1.chr20.fastq.gz | sed -n '2~4p' | head -n 10000 | grep -o "ATGGGA"
```

Continuing to build our command, we can pipe the output of the `grep` command to the `wc -l` command to count the number of times this match was recovered. 
```bash
# Count how many times in the first 10000 FASTQ our (pretend) adapter sequence occurs
zcat SRR1039508_1.chr20.fastq.gz | sed -n '2~4p' | head -n 10000 | grep -o "ATGGGA" | wc -l
```
Just to break down this fairly complex command into a simpler format:
1. `zcat SRR1039508_1.chr20.fastq.gz` prints the contents of the file to the screen
2. `sed -n '2~4p'` prints the second line and skips four lines and prints the line again
3. `head -n 10000` limits the output to the first 10000 sequence lines
4. `grep -o "ATGGGA"` looks for our adapter pretend adapter sequence in the first 10000 reads and prints the pattern to the screen when found
5. `wc -l` counts the number of lines that are printed to the screen


By adjusting just a couple of these steps we can instead count all of the instances of individual DNA bases (A,T,C,G) called by the sequencer in this sample. Here we use the `sort` command to sort the bases printed by `grep` and then use the `uniq` command with the `-c` option to count up the unique elements.

1. `zcat SRR1039508_1.chr20.fastq.gz` prints the contents of the file to the screen
2. `sed -n '2~4p'` prints the second line and skips four lines and prints the line again
3. `head -n 10000` limits the output to the first 10000 sequence lines
4. `grep -o .` looks for any single pattern (A, T, C, G, N) and prints it on a separate line
5. `sort` sorts the bases alphabetically
6. `uniq -c` counts the instances of each unique element

```bash
# Determine the G/C content of the first 10000 reads
zcat SRR1039508_1.chr20.fastq.gz | sed -n '2~4p' | head -10000 | grep -o . | sort | uniq -c
```

Now we have the frequency of each nucleotide from the first 10,000 records. A quick and easy program to get GC content, (you can see that there is around 52% GC content just by comparing the A/T count to the GC count). GC content is used in basic quality control of sequence from FASTQs to check for potential contamination of the sequencing library. We just used this code to check 1 sample, but what if we want to know for our 4 samples or 100 samples?

## For & while loops
-----

Loops allow us repeat operations over a defined variable or set of files. Essentially, you need to tell BASH what you want to loop over, and what operation you want it to do to each item.

Similar to how we set variables in our environment, here we use the variable `i`, set in the conditions of the loop, to reference all the elements to be looped over in the operation using `$i` in the `for` loop example below. The syntax of a for loop are essentially `for CONDITION; do CODE;done`, with the conditions and code punctuated by a semicolon.

```bash

# loop over numbers 1:10, printing them as we go
for i in {1..10}; do echo "$i"; done

```

This loop essentially states `for` i in the list of numbers from 1 to 10, `do` print `$i` to the screen using the `echo` command, finish the loop `done` when you get to the bottom of my list (10). 

Alternatively, if you do not know how many times you might need to run a loop, using a `while` loop may be useful, as it will continue the loop until the Boolean (logical) specified in the first line evaluates to `false`. An example would be looping over all of the files in your directory to perform a specific task. e.g.

```bash
ls *.fastq.gz | while read x; do \
   # tell me what the shell is doing
   echo $x is being processed...\n;
   # unzip w/ zcat and print head of file
   zcat $x | head -n 4;  \
   # print 3 lines to for ease of viewing
   echo -e "\n\n\n" ;
done
```
This loop is a bit more complex and you will notice each new line of code within the loop ends in `; \` to indicate that even though the codes starts on a new line (for readability) we are still entering commands to be executed within the loop. In this loop we are saying:

1. `ls *.fastq.gz` list all the files in the current directory ending in *.fastq.gz* 
2. `while read x; do` as long as there are still files in the list save the filename as $x
3. `echo $x is being processed..\n;` print the sample name to the screen
4. `zcat $x|head -n 4` zcat the file and print the first entry
5. `echo -e "\n\n\n" ;` print three new lines to separate the output from the previous file (the -e flag enables interpretation of `\`) 

Perhaps we want to check how many of the first 10000 reads in each file contain the start codon `ATG`. We can do this by searching for matches and counting how many times it was found, and repeating this process for each sample using a while loop.

```bash
ls *.fastq.gz | while read x; do \
   echo $x
   zcat $x | sed -n '2~4p' | head -n 10000 | grep -o "ATG" | wc -l
done
```

We could use one of these loops to perform the nucleotide counting task that we performed on a single sample above, but apply it to all of our samples in a single command.

```bash
ls *.fastq.gz | while read x; do \
   echo -e "\n"
   echo processing sample $x
   zcat $x | sed -n '2~4p' | head -10000 | grep -o . | sort | uniq -c ;
done
```

## Scripting in bash
----

So loops are pretty useful, but once we write some useful code we want to save it to use later. This way we aren't reinventing the wheel and we can share the program we just wrote with other lab members.

One way to do this would be to write this series of commands into a Bash script, that can be executed at the command line, passing the files you would like to be operated on to the script. Though not required, generally the suffix of a script indicates the language the script is written in, for BASH scripts we use `*.sh`, for python we use `*.py`, for perl we use `*.pl`, and for R we use `*.R`. 

Lets create a script to count nucleotide frequencies:

```bash
nano count_ATGC.sh
```
The first thing we need to add to our script (and this is true of any script) is called a shebang, this line indicates the coding language used in the body of the script. The BASH shebang is `#!/bin/bash`.

Next we add our program to the script. As in the loops we use the `$` to specify the input variable to the script. `$1` represents the variable that we want to be used in the first argument of the script. Here, we only need to provide the file name, so we only have 1 `$`, but if we wanted to create more variables to expand the functionality of our script, we would do this using `$2`, `$3`, etc.


Copy the following code into the nano editor file you just opened and use the ctrl+x command to close the file and save the changes you made.
```bash
#!/bin/bash
echo processing sample "$1"; zcat $1 | sed -n '2~4p' | head -n 10000 | grep -o . | sort | uniq -c
```

Now run the script, specifying the a FASTQ file as variable 1 (`$1`)

```bash
# have a quick look at our script
cat count_ATGC.sh

# now run it with bash - which again indicates that the code is in the BASH language
bash count_ATGC.sh SRR1039508_1.chr20.fastq.gz
```

If we wanted to run on multiple samples we can use our while loop again to do this for all the FASTQs in our directory
```bash
ls *.fastq.gz | while read x; do \
   bash count_ATGC.sh $x
done
```

What if we wanted to write the output into a file instead of printing to the screen? We could save the output to a file that we can look at, save to review later, and document our findings. The `>>` redirects the output that would print to the screen to a file.
```bash
# create the text file you want to write to
touch out.txt

# run the loop
ls *.fastq.gz | while read x; do \
   bash count_ATGC.sh $x >> out.txt
done

# view the file
cat out.txt
```

These example programs run fairly quickly, but stringing together multiple commands in a bash script is common and these programs can take much longer to run. In these cases we might want to close our computer and go and do some other stuff while our program is running.

We can do this using `nohup` which allows us to run a series of commands in the background, but disconnects the process from the shell you initially submit it through, so you are free to close this shell and the process will continue to run until completion.
```bash
# run your GC content program using the executable you just made
nohup bash count_ATGC.sh SRR1039508_1.chr20.fastq.gz &

# print the result
cat nohup.out
```

## Quality control of FASTQ files
----

While the value of these exercises may not be immediately clear, you can imagine that if we wrote some nice programs like we did above, and grouped them together with other programs doing complimentary tasks, we would make a nice bioinformatics software package. Fortunately, people have already started doing this, and there are various collections of tools that perform specific tasks on FASTQ files.

One excellent tool that is specifically designed assess quality of FASTQ file is [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/). FastQC is composed of a number of analysis modules that calculate QC metrics from FASTQ files and summarize the results into an HTML report, that can be opened in a web browser.

>Checking quality of raw NGS data is a <u>key step</u> that should be done <u>before</u> you start doing any other downstream analysis. In addition to identifying poor quality samples, the quality control assessment may dictate <u>how</u> you analyze your data downstream.  

Lets have a look at some example QC reports from the FastQC documentation:

- [Good Illumina Data FastQC Report](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/good_sequence_short_fastqc.html)
- [Bad Illumina Data FastQC Report](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/bad_sequence_fastqc.html)

Run FASTQC on our data and move the results to a new directory.
```bash
# specify the -t option for 4 threads to make it run faster
fastqc -t 1 *.fastq.gz

# move results to a new folder
mkdir $FOB/rawQC
mv *fastqc* $FOB/rawQC

# move into it and ls
cd $FOB/rawQC
ls -lah
```

**Note**: FastQC does not use the entire dataset, just the first few thousand reads in the FASTQ file, therefore there could be some bias introduced by this, although we assume there isn't since entries are placed into FASTQ files randomly.

Opening and evaluating an individual HTML file for each FASTQ file is tedious and slow. Luckily, someone built a tool to speed this up. [MultiQC](https://multiqc.info/) searches a specified directory (and subdirectories) for log files that it recognizes and synthesizes these into its own browsable, sharable, interactive .html report that can be opened in a web-browser. *MultiQC* recognizes files from a very wide range of bioinformatics tools (including FastQC), and allows us to compare QC metrics generated by various tools across all samples so that we can analyze our experiment as a whole.

Lets run MultiQC on our FastQC files:
```bash
multiqc .
```

Copy to report to your **LOCAL MACHINE** in a new folder and open in a web-browser:
```bash
# make a directory and go into it (ON YOUR LOCAL MACHINE)
mkdir fund_of_bioinfo/
cd fund_of_bioinfo/

# use secure copy (scp) to download the files to your local machine - remember to change the netID before you to your own, or your initials, and paste this command into the terminal
scp netID@discovery7.dartmouth.edu:/dartfs-hpc/scratch/NETID/fundamentals_of_bioinformatics/rawQC/multiqc_report.html .
```

You can find the MultiQC report run on the complete dataset across all samples in the dataset in the Github repository, under `QC-reports`. Let's open it and explore our QC data. If the `scp` command did not work for you there is a copy of the multiqc report in the Github repo you downloaded under `Day-1/data/multiqc_report.html`.


## Read pre-processing & trimming
------

An additional QC step one should perform on raw FASTQ data is to *pre-process* or *trim* the sequences to remove sequences that we are not interested in, or were not called confidently by the sequencer.

This step is **optional** in most analysis, although should be based on an empirical decision that leverages the QC assessment of raw FASTQs using a quality report like the one we just generated with FASTQC/MultiQC. For example, if we see we have a large number of adapter sequences in our data, or a high proportion of low-quality bases near our read ends, we may wish to trim our raw reads. Otherwise, we could skip this step in the analysis.

Notably, some read mappers account for mismatches or low quality bases at the end of reads in a process called *soft-clipping*, where these bases are masked from being included in the alignment, but are technically still part of the sequence read in the FASTQ. If you are using an aligner that performs soft-clipping, you could consider omitting read trimming of FASTQ files.

### Motivation for read trimming: Downstream steps are more efficient

Several algorithms exist for trimming reads in FASTQ format. Generally, these algorithms work by looking for matches to the sequence you specify at the 5' and 3' end of a read. You can specify the minimum number of bases you would like to be considered a match, as the algorithm will trim partial matches to the sequence you specify. Examples of sequences you might want to remove include:  
- adapter sequences  
- polyA tails   
- low quality bases  

<p align="center">
<img src="../figures/read_processing.png" title="xxxx" alt="context"
	width="70%" height="87%" />
</p>

### Read trimming with cutadapt

[Cutadapt](https://cutadapt.readthedocs.io/en/stable/) is a useful tool for cleaning up sequencing reads, allows for multiple adapters to be specified simultaneously, and has an array of options that can be tweaked to control its behavior.

Basic usage of cutadapt:
```bash
cutadapt -a ADAPTER -g ADAPT2 [options] -o output.fastq input.fastq.gz
```
- `a` specifies an adapter to trim from the 3' end of read 1
- `g` specifies an adapter to trim from the 5' end of read 1
- `o` specifies name of out file

For paired-end reads:
```bash
cutadapt -a ADAPT1 -g ADAPT2 [options] -o out1.fastq.gz -p out2.fastq input1.fastq.gz input2.fastq.gz
```
Capital letters are used to specify adapters for read 2.

If we wanted to trim polyA sequences, as we often do in RNA-seq, and save the output to a report called cutadapt.logout, we could use:  
```bash
cutadapt -a 'A{76}' -o out.trimmed.fastq.gz input.fastq.gz > cutadapt.logout;
```
`-a A{76}` tells cutadapt to search for stretches of A bases at the end of reads, with a maximum length of the read length (76bp).

Since the polyA and adapter sequence contamination is relatively low for this dataset, we won't trim any specific sequences, although we will perform basic quality and length processing of the raw reads. Lets make a new directory and do this for do this for one sample.
```bash
mkdir -p $FOB/trim
cd $FOB/trim

cutadapt \
   -o SRR1039508_1.trim.chr20.fastq.gz \
   -p SRR1039508_2.trim.chr20.fastq.gz \
   $FOB/raw/SRR1039508_1.chr20.fastq.gz $FOB/raw/SRR1039508_2.chr20.fastq.gz \
   -m 1 -q 20 -j 4 > SRR1039508.cutadapt.report
```

- `-m` removes reads that are smaller than the minimum threshold
- `-q` quality threshold for trimming bases
- `-j` number of cores/threads to use

You should now have a trimmed FASTQ file in this directory that can be used for an alignment. Lets look at the report that cutadapt generated.
```bash
cat SRR1039508.cutadapt.report
```

Now lets run this on multiple samples:
```bash 
ls $FOB/raw/*.chr20.fastq.gz | while read x; do \

   # save the file name
   sample=`echo "$x"` 
   # get everything in file name after "/" and before "_" e.g. "SRR1039508"
   sample=`echo "$sample" | cut -d"/" -f7 | cut -d"_" -f1` 
   echo processing "$sample"

   # run cutadapt for each sample 
   cutadapt \
      -o ${sample}_1.trim.chr20.fastq.gz \
      -p ${sample}_2.trim.chr20.fastq.gz \
      $FOB/raw/${sample}_1.chr20.fastq.gz $FOB/raw/${sample}_2.chr20.fastq.gz \
      -m 1 -q 20 -j 4 > $sample.cutadapt.report
done
```

You should now have trimmed FASTQ files in this directory that we will use for the alignment. You should also be able to see and print each of your reports from cutadapt. 
```bash
ls *cutadapt.report | while read x; do
   echo -e "\n\n"
   echo Printing $x
   echo -e "\n"
   cat $x
done
```

**Additional note:** For data generated at Dartmouth, since much of the data in the Genomics core is generated using an *Illumina NextSeq 500*, we also often use the `--nextseq-trim` option in cutadapt.

This option works in a similar way to the quality threshold option `-q` BUT ignores Q-scores for stretches of G bases, as some Illumina instruments, such as the NextSeq, generate strings of Gs when the sequencer 'falls off' the end of a fragment and dark cycles occur, and therefore provides more appropriate quality trimming for data generated on these instruments.

### Break out exercises

- Run through the commands above to generate a quality report and trim the reads

- What do we think about the quality of our dataset?

- How would this affect the flags you might choose to use when preprocessing the data?
   - higher or lower quality threshold?
   - leave off the quality filter?
   - adjust the minimum read size threshold?

- Can you write a loop to trim a series of fastq files and save it to a bash script?


--------
If you didn't have time to finish the lesson or you got lost you can copy the files needed for the next step using the following command. 

```bash

# IF YOU DIDN'T HAVE TIME TO FINISH TRIMMING COPY THOSE FILES NOW
mkdir -p $FOB/trim
cp $SOURCE/trim/* $FOB/trim/
```
