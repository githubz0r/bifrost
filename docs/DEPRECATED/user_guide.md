# User Guide

This is the basic folder structure with a sample run containing 5 samples:

```bash
├── config
│   ├── config.yaml
│   └── keys.txt
├── input
│   └── 2019
│       └── example_run_01
│           ├── Sample1_S1_L001_R1_001.fastq.gz
│           ├── Sample1_S1_L001_R2_001.fastq.gz
│           ├── Sample2_S2_L001_R1_001.fastq.gz
│           ├── Sample2_S2_L001_R2_001.fastq.gz
│           ├── Sample3_S3_L001_R1_001.fastq.gz
│           ├── Sample3_S3_L001_R2_001.fastq.gz
│           ├── Sample4_S4_L001_R1_001.fastq.gz
│           ├── Sample4_S4_L001_R2_001.fastq.gz
│           ├── Sample5_S5_L001_R1_001.fastq.gz
│           └── Sample5_S5_L001_R2_001.fastq.gz
├── output 
    └── 2019
        └── example_run_01 
            │   # (before bifrost is run)
            ├── samples -> ../../../input/2019/example_run01/
            ├── sample_sheet.tsv
            │── src # This is a copy of the bifrost folder.
            │   # (additional files after software is run)
            ├── bifrost
            ├── bifrost_successfully_initialized_on_(DATETIME)
            ├── components
            ├── config.yaml
            ├── Sample1
            ├── Sample2
            ├── Sample3
            ├── Sample4
            ├── Sample5
            └── run_cmd_bifrost.sh

```

`config` contains `config.yaml` which includes user specific config values to run the pipeline such as the group name for the torque queue and which components to run. This is also pointed to by the Webserver for access to the bifrost web interface.

## Before starting

Create a `.condarc` in your home directory with the following content so that the bifrost environment can be run:

```
envs_dirs:
  - /home/projects/ssi_10003/apps/anaconda/envs

pkgs_dirs:
  - /home/projects/ssi_10003/apps/anaconda/pkgs
```

## Basic workflow

The basic workflow starts when the user adds a new directory (considered a 'run') to the input/year directory.
This run directory should contain the read pairs with names matching the regex specified in the config file. The default name from illumina MySeq or NextSeq machines works. An example_run_01 is included to help you get started.

The next step is either triggered by the user or, when implemented, executed by a chron job.

## Setting up the run
(All of the following steps have already been done for you for example_run_01)
- Create a directory in output/year (we'll call it output/year/run, an example could be output/2019/RUN2019001) with the same name as the one created in input/year (input/year/run)
- Symlink input/year/run into output/year/run/samples
- Copy recursively the source code located in /home/projects/ssi_10003/apps/bifrost-private as output/year/run/src
- If you have a sample sheet with sample metadata, add it as output/year/run/sample_sheet.tsv (or sample_sheet.xslx, but you need to set config to sample_sheet=sample_sheet.xlsx)

### Sample sheet

You can use /output/2019/example_run_01/sample_sheet.tsv as an example. It should have a header and the order is not important as it is being read by pandas. The fields are:

**SampleID:** Mandatory. This name is mapped in the database to sample_name, but our code currently depends on that name instead of the mapped one as an oversight. I'll add that to the task list so in the future it will be sample_name (or whatever field you configure in config.yaml).

**group:** Recommended. We use it for the different supplying labs and it can be used to sort samples in the web reporter, but it is not required if it doesn't really apply to your case.

**provided_species:** Strongly recommended. We detect the species anyway, but it is useful for QC. The current behavior is that qc will flag it if missing, or if it is different than the detected one (to identify contaminations or wrongly identified species). The QC behavior can be changed of course.

**Comments:** Optional. This value is shown in the reporter, so don't store anything here that you don't want anyone with access to the web reporter to see. In our case, it is empty in most samples but we use it for library preparation failures and misc information. As with SampleID, we will change the name to comments to have a consistent naming scheme.

You can add any extra values to the sample sheet and they will not be shown in the reporter, but will be stored in the database (in the sample object, under sample_sheet).

Also note, to make it work with the xlsx you need to change the default config.yaml value for sample_sheet to `sample_sheet=sample_sheet.xlsx`.

## Running bifrost

- Load the environment:

```bash
source activate bifrost
```

- Set the location of the MongoDB key file. The file should be named keys.txt and contain the mongo URI for the KMA

```bash
export BIFROST_DB_KEY=/home/projects/KMA_project/data/bifrost/config/keys.txt
```

- Run bifrost from /output/year/run with the following command:

```bash
snakemake -s src/bifrost.smk --configfile /home/projects/KMA_project/data/bifrost/config/config.yaml
```

You can check the running status in your run checker.

The output for each sample will be generated in output/year/run/samplename

## Maintenance

### Rerunning a component in a sample

Go into the sample folder (the one you want to run), open cmd_bifrost.sh and copy the line that starts with snakemake and includes the source code of the component you wan to rerun. You can run that in the terminal directly and it will run in your node.

You can also create a new batch file with the same header as cmd_bifrost.sh and submit it to the queue system.

### Removing a run

This task should be done in two steps. First, activate the bifrost environment, make sure that you have the correct key path in the BIFROST_DB_KEY variable and launch a python interpreter. Run the following command to delete the run from the database.

```python
from bifrostlib import datahandling
datahandling.delete_run(name="runname")
```

Note that delete_run can be run with the run_id instead by using `datahandling.delete_run(run_id="runidstring")`.

Now that the data has been removed from the database, you can delete the data files from the filesystem.

## Debugging

If there are any problems running the pipeline contact Martin (mbas@ssi.dk) or Kim (kimn@ssi.dk). However, you can access some logs if you want to get more detail on where the program is failing.

There are two main stages in the execution: Creating the run and running the components.

### Creating the run

If you can't find the run in the run checker, bifrost probably hasn't finished creating it. Another way to confirm is to go to a sample folder (if they exist) and check that there are no component folders created yet.

The logs for this stage can be found in output/year/run/.snamemake/log/ and in output/year/run/bifrost/log

### Running the components

If a component fails, (it shows as Failed or in rare cases as Running for a long time, in the Run checker), you can check the logs in `output/year/run/<sample>/<component>/log/`

There might be some logs from torque in your home directory, although we plan to change it to a proper location.

