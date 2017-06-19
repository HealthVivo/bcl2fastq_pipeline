For a version supporting bcl2fastq V1 from Illumina, see the "[bcl2fastqV1](https://github.com/maxplanck-ie/bcl2fastq_pipeline/tree/bcl2fastqV1)" branch.

This is our bcl to fastq pipeline. Features include:

  * Runs as a background process, rather than a cron job that needs to check if another instance is already running
  * Handles undetermined indices in a more coherent manner
  * Adds quality metrics to the report email
  * Performs a contamination screen
  * Automatically delivers data to most groups and posts external data to the F\*EX server
  * Internal groups additionally have their data linked into Galaxy, if possible
  * Compiles an automated project-report (pdf), to go along with each submission. This uses [ReportLab](https://pypi.python.org/pypi/reportlab).
  * Functions divided into more meaningful subfiles in a module, rather than being scattered across levels of shell scripts
  * Written explicitly for python3, just to future proof things a bit.

General workflow
================
The general workflow of this pipeline is as follows:
  1. Read in `bcl2fastq.ini` from `~/`. Note that this file can be changed while the program is running.
  2. Look for new flow cells.
     1. Iterate through directories listed under `[Paths]`->`baseDir`, looking for the special `RTAComplete.txt` file.
        * This program is currently hardcoded to look only in flow cells generated by machines SN7001180 and NB501361.
     2. Check for the number of sample sheets (`SampleSheet*.csv`), since a separate folder will be output per sample sheet
     3. For each sample sheet (or directory, if there are no sample sheets), check to see if it has already been processed.
        * A flow cell is marked as being processed if:
           1. There is an equivalent directory under `[Paths]`->`outputDir`, with a possible `_lanes_1_2` suffix (or something equivalent, for different lanes)
           2. That directory contains a file named `casava.finished` (our old pipeline) or `fastq.made` (the current program)
           3. This sets the `[Options]`->`runID` field in the configuration file.
  3. If there are no new flow cells to process, the program sleeps (the duration is set in `[Options]`->`sleepTime`)  and starts again at step 1.
  4. If there are new flow cells to process, ensure that there is sufficient space in `[Paths]`->`outputDir`. This is set in `[Options]`->`minSpace`.
     * Note that having insufficient space will lead to an email being sent to addresses set in `[Email]`->`errorTo`. The program will then sleep (see step 3) and loop (i.e., go back to step 1).
  5. Assuming there is at least one new flow cell and there's sufficient space, the program will generate fastq files.
     1. The sample sheet is first rewritten to strip out illegal character (e.g., anything with an umlaut). The rewritten sample sheet is placed in `/tmp` and not removed after running.
     2. The barcode masking strategy is inferred from `RunInfo.xml`, unless it's already specified in the config file.
     3. The program specified via `[bcl2fastq]`->`bcl2fastq` is run with options specified in `[bcl2fastq]`->`bcl2fastq_options`. In addition to these options, the follow are hard coded:
        1. `-o outputDir/runID`: The output directory is set to `[Paths]`->`outputDir/runID`. This directory is created if it doesn't already exist.
        2. `-r runDir/runID`: This is the directory that's being processed (`[Paths]`->`runDir`/`runID`).
           * This directory may be read only!
        3. `--interop-dir seqFacDir/runID/InterOp`: This prevents `bcl2fastq` from attempting to write to the running directory, which could be dangerous. See `[Paths]`->`seqFacDir` for where this is.
           * The sequencing facility has requested this directory. Note that the path will be created if it doesn't already exist.
        4. The file named `bcl.done` in the output directory is touched. If the pipeline experiences an error and restarts then it will then skip the already completed demultiplexing step.
  6. Files and directories are renamed for consistency with previous data produced at the institute.
      * `files.renamed` is then touched (if it already exists then this step will be skipped)
  7. A number of "post make" steps are run. This terminology is a hold-over from the previous bcl2fastq pipeline, which used `make` to generate the fastq files.
     1. If a flow cell was run on the HiSeq 3000, optical duplicates are removed and placed in a separate file with clumpify.sh from bbmap.
        * This has multiple workers, each of which is multithreaded. This is due to the program not nicely respecting thread settings and occasionally requesting gobs of memory.
        * See `[Options]`->`deduplicateInstances` for the number of simultaneous instances.
        * See `[bbmap]` for other related options
        * This step creates a ".duplicate.txt" file for each sample. If the pipeline later experiences an error and sees such a file then this step will be skipped for the given sample (the step is resource intensive).
     2. FastQC is run on each output fastq file.
        * This is run in a multithreaded manner, see `[Options]`->`postMakeThreads` for the number of workers.
        * See options under `[FastQC]` for executable paths and options.
        * The output is placed in `[Paths]`->`outputDir`/`runID`/FASTQC_project_name.
     3. An md5sum is made of the fastq files in each project (see the file named "md5sums.txt").
        * As with FastQC, this is multithreaded, with the number of workers threads set via `[Options]`->`postMakeThreads`.
     4. A contamination screen is run with fastq_screen after downsampling read #1 of each sample.
     5. Runs multiQC on the output of FastQC.
     6. Additional steps can be added to `afterFastq.py`, though note that the package will need to be reinstalled and the process restarted.
  8. Xml files and FastQC outputs are copied to a location readable by the sequencing facility.
     * This is location is set via `[Paths]`->`seqFacDir` and things placed under a `runID` subdirectory, as was the case with `InterOp` above.
     * Currently, the xml files are `RunInfo.xml` and `runParameters.xml`.
  9. A summary PDF file is created for each of the projects. All of the metrics from this are gathered from `Stats/ConversionStats.xml`.
     * Everything about these PDFs is hard-coded. In an ideal world, this would have some sort of plugin interface.
  10. FastQC and fastq files are copied to the group directories, under `sequencing_data/`.
      * If the directories already exist then an error is produced. This is to ensure that nothing is inadvertently over-written!
      * Only projects starting with the letters "A" or "C" will be distributed. Those starting with "A" are distributed to the groups and those with "C" to Andreas (`[Paths]->DEEPDir`).
      * Projects starting with "B" are uploaded to the F\*EX server and an email with the link sent to "Uni"->"default" or "Uni"->"Schuele". The latter only occurs for DEEP data from the Scheule group.
      * Projects starting with "A" are linked into Galaxy, if and only if the associated group has a data library with a "sequencing data" folder.
      * Output directories and files for projects starting with "A" have their permissions changed to ensure that groups do not have write access.
  11. A summary email is produced (largely by parsing `Stats/DemultiplexingStats.xml`) and sent to the email addresses specified via `[Email]`->`finishedTo`.
      * Note the other options under `[Email]`, which specify the host name of the outgoing email server and the outgoing email address.
  12. A file named `fastq.made` is produced in `[Paths]`->`outputDir`/`runID`/.

Special files
=============

The following files have special meanings if found in an output directory:

 * `bcl.done`: Demultiplexing has finished (touch this if you run it manually)
 * `files.renamed`: The fastq files and directories have been renamed to have things like `Project_` and `Sample_` prepended and "_001" stripped.
 * `*.duplicate.txt`: Produced by clumpify. If it exists then clumpify won't be run
 * `fastq.made`: The flow cell is finished

Configuration file
==================
The configuration file is a human readable text file named `bcl2fastq.ini` and must be placed in the home directory (`~/`) of the user running this package. Currently, the file has the following sections:
  * `[Paths]` - Holds all path information
    * `baseDir` - The base directory where the HiSeq writes its output
    * `outputDir` - The base directory where the demultiplexed fastq files and fastQC/md5sum output should be written.
    * `seqFacDir` - The base directory readable by the sequencing facility, for the files they're interested in.
    * `groupDir` - The base directory holding all group's datasets (currently, this should be `/data` for us).
    * `DEEPDir` - The base directory holding all DEEP datasets.
    * `logDir` - The demultiplexing log for each run is written here.
  * `[FastQC]`
    * `fastqc_command` - Either just `fastqc` or possibly the full path, as appropriate.
    * `fastqc_options` - Options for fastqc
  * `[MultiQC]`
    * `multiqc_command` - Path to the multiqc command
    * `multiqc_options` - Options for multiqc
  * `[fastq_screen]`
    * `fastq_screen_command` - The command to run `fastq_screen`
    * `fastq_screen_options` - Options to be passed to `fastq_screen`
    * `seqtk_command` - The path to SeqTK, which is used for downsampling
    * `seqtk_options` - Options given to SeqTK, typically the seed (e.g., `-s 123456`)
    * `seqtk_size` - The target number to downsample to (e.g., `1000000`)
  * `[bcl2fastq]`
    * `bcl2fastq` - Either just `bcl2fastq` or pissibly the full path, as appropriate.
    * `bcl2fastq_options` - The options for `bcl2fastq`. Something like `--use-bases-mask Y\*,I6n,Y\* -l WARNING --barcode-mismatches 0 --no-lane-splitting` is recommended.
  * `[Options]` - These are more generic options that don't fit elsewhere.
    * `index_mask` - The index mask (`--use-bases-mask`) given to `bcl2fastq`. This often needs to be changed every few runs, since most of the time it's `I6n`, but not always.
    * `postMakeThreads` - After the fastq files are made, things like fastqc are run on each of them. This value sets the total number of worker threads that are used to do that.
    * `minSpace` - The minimum free space (in gigabytes) that must be free in the `outputDir`. Having less free space than this results in an error email message.
    * `sleepTime` - The amount of time the programs sleeps before restarting (in hours). Importantly, if something is broken and error emails begin to be sent then this also specifies how frequently they'll be produced.
    * `runID` - This should be left blank.
    * `sampleSheet` - This should be left blank.
  * `[Email]`
    * `errorTo` - A comma-separated list of email addresses to which error reports should be sent.
    * `finishedTo` - A comma-separated list of email addresses to which reports of finishing a flow cell should be sent.
    * `fromAddress` - The email address from which emails are sent.
    * `host` - The outgoing email server.
  * `[Uni]`
    * `default` - The email address that F\*EX should send an email to when a "C" project is uploaded (except for the Scheule group).
    * `Scheule` - The email address that F\*EX should send an email to when a "C" project from the Scheule group is uploaded.
  * `[Version]` - Everything under here is included in the PDF files generated for each project.
    * `pipeline` - A version number for this package
    * `bcl2fastq` - The version number of bcl2fastq from Illumina
    * `fastQC` - The version number of fastQC.
  * `[Galaxy]`
    * `API key` - The API key to use when contacting the Galaxy server. DO NOT SHARE THIS!
    * `URL` - The galaxy server's URL (e.g., https://usegalaxy.org, though that'd obviously not work)

A few general notes are in order:
  * Blank lines my be added pretty much anywhere.
  * Comments need to be on separate lines and can be preceded by either `#` or `;`.
  * The order of things doesn't matter.
  * Quotes should not be used! `fastqc=/usr/bin/fastqc` is not the same as `fastqc="/usr/bin/fastqc"`!
  * All mentioned settings **must** be present! There's currently no method to support skipping steps if a line is blank or absent!

Dependencies
============
This package has the following dependencies:
  * Python3 (python2 will explicitly not work, since some package and function names differ).
  * The configparser module
  * The reportlab module
  * bioblend
  * numpy and matplotlib
  * bcl2fastq version 2+
  * fastq\_screen
  * seqtk
  * FastQC must be present
  * MultiQC must be present
  * md5sum must be present
  * The Pillow python module must be relatively up to date and functional (can't install in Ubuntu and have it work in CentOS).
  * There must be an available sendmail server somewhere. This package currently does not support authentication, but that could presumably be added.
  * pigz
  * splitFastq, which comes in this repository but must be compiled manually
