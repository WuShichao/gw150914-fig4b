# Reproducing the Detection of GW150914: the first observation of gravitational waves from a binary black hole merger

**Duncan A. Brown<sup>1</sup>, Karan Vahi<sup>2</sup>, Michela Taufer<sup>3</sup>, Von Welch<sup>4</sup>, Ewa Deelman<sup>2</sup>**

**<sup>1</sup>Syracuse University**

**<sup>2</sup>University of Southern California**

**<sup>3</sup>University of Tennessee Knoxville**

**<sup>4</sup>Indiana University**


## License

![Creative Commons License](https://i.creativecommons.org/l/by-sa/3.0/us/88x31.png "Creative Commons License")

This work is licensed under a [Creative Commons Attribution-ShareAlike 3.0 United States License](http://creativecommons.org/licenses/by-sa/3.0/us/).

## Abstract

In February 2016, LIGO announced the first observation of gravitational waves from merging black holes, known as GW150914. The event was first detected by a low-latency computational search that identifies candidate events, but does not provide a final estimate of the statistical significance. To establish the confidence in the detection large-scale scientific workflows were used to measure the event's significance and establish the detection confidence. These workflows used code written by the LIGO Scientific Collaboration and were executed across a range of cyberinfrastructure resources. The code to perform these analyses are publically available, but there has not yet been an attempt to directly *reproduce* the results, although several subsequent analyses have *replicated* the analysis, confirming the detection. To study the reproducability of a major scientific discovery, we attempt to reproduce the result from the compact binary coalescence search presented in the GW150914 discovery paper using publicly available code executed primarily on the Open Science Grid.

## Data Products

The main script for reproducing the LIGO/Virgo PyCBC GW150914 analysis is [generate_workflow.sh](https://github.com/duncan-brown/gw150914-fig4b/blob/master/generate_workflow.sh). This script performs the following actions:

 1. Install a version of PyCBC that contains the tools needed to obtain data from the Gravitational Wave Open Science Center (GWOSC).
 2. Create a wrapper script `pycbc_losc_segment_query.sh` than has the same command line API as the LIGO DQSEGDB tools, but retrieves the metadata from GWOSC.
 3. Create a wrapper script `minifollowup_wrapper.sh` that allows the PyCBC v1.3.2 follow-up workflows to be run using Pegasus WMS 4.9.
 4. Download the bundled executables for the codes that create and run the workflow.
 5. Download the configuration file that contains the locations of the bundled exectables that are executed in the workflow and modify this file to download them from the cache on https://pegasus.isi.edu rather than the original (defunct) LIGO location.
 6. Set the `LIGO_DATAFIND_SERVER` environment variable to a server that indexes the GWOSC frame files stored in CVMFS.
 7. Set the `LAL_DATA_PATH` environment variable to use data stored under CVMFS.
 8. Run the workflow generation script with a set of `--config-overrides` that switch the workflow to:
   * Use the `minifollowup_wrapper.sh` to plan the follow-up sub-workflow.
   * Perform the metdata segment query with the wrapper script `pycbc_losc_segment_query.sh`.
   * Used data stored in the channel `GWOSC-16KHZ_R1_STRAIN` from the frame type to `H1_LOSC_16_V1` GWOSC frames.
   * Configure the segment generation code to use the GWOSC segment type `H1:RESULT:1` and `L1:RESULT:1`.
   * Use the dummy veto definer file `H1L1-DUMMY_O1_CBC_VDEF-1126051217-1220400.xml` since the GWOSC segment wrapper obtains `SCIENCE-CAT1` analysis and `CAT2` veto segments from the GWOSC data.
   * Set the `segments-database-url` to the GWOSC server and use the segment files generated by `pycbc_losc_segment_query.sh` rather than re-generating them by setting the `segments-generate-segment-files:if_not_present` flag.
   * Use `/bin/true` as the `segments_from_cats` executable, as this code is not needed.
 9. Fix `pegasus.dir.storage.mapper.replica.file` in the sub-workflows for compatibility with Pegasus 4.9 and OSG execution.
 10. Update the workflow to indicate that the GWOSC frame files under CVMFS are available on the OSG.
 11. Update the main workflow so that Pegasus is run with the optionm `--staging-site osg=local` when generating sub-workflows.
 12. Run `pycbc_submit_dax` to plan and execute the workflow.
   
The second script [make_pycbc_hist.sh](https://github.com/duncan-brown/gw150914-fig4b/blob/master/make_pycbc_hist.sh) creates PyCBC environment that can be used to run the program [pycbc_dogsin_hist_sigmas_arrow](https://github.com/duncan-brown/gw150914-fig4b/blob/master/pycbc_dogsin_hist_sigmas_arrow) that makes the result plot. It should be run with no arguments in the directory where the workflow `output` directory has been created.
 
## Reproducibility Notes

### Datafind Server

The PyCBC workflow queries a LIGO Datafind Server to map metdata queries (time ranges and data types) into file locations. Running the workflow generation script requires the environment variable `LIGO_DATAFIND_SERVER` to be set to a server that indexes the GWOSC data. The script currently queries a public server at Syracuse University that indexes the GWOSC data from the LIGO/Virgo O1 and O2 runs in CVMFS. This server can be used to run the workflow on e.g. the OSG and access the GWOSC data via CVMFS.

For users who wish to store data in a different location, or maintain their own datafind server we provide [RPMs](https://github.com/duncan-brown/gw150914-fig4b/blob/master/rpms) for installation of the LDAS Diskcache API (thai indexes the data) and the LIGO Datafind Server (that responds to metadata queries from the workflow) on a CentOS 7 machine. We also provide an LDAS Diskcacge API configuration file [diskcache.rsc](https://github.com/duncan-brown/gw150914-fig4b/blob/master/diskcache.rsc) and a Datafind server configuration file [datafind-server.ini](https://github.com/duncan-brown/gw150914-fig4b/blob/master/datafind-server.ini) that can be used to index the GWOSC data in CVMFS.

### System Setup

Running the workflow requires a system with HTCondor and Pegasus WMS 4.9 installed. The compute-intensive jobs can be run on the Open Science Grid, if the HTCondor submit host is configured to allow jobs to flock to OSG. Large memory machines are needed for the post processing jobs, as described in the paper.

Some other things to keep in mind

1. The repository should be cloned to a shared fileysystem space on your local cluster. The directory where you clone the repository should be accessible on the nodes making up your local HTCondor Pool.
2. The inspiral jobs are setup to run on OSG resources. For that the Pegasus workflows will be setup to transfer intermediate data and outputs to/from OSG using the SCP endpoint on your submit host.
   * For SCP transfers, the Pegasus workflows will use the SSH key at this location ${HOME}/.ssh/workflow . We recommend you generate a new passwordless key and use it for this workflow.
   * For the jobs to run on OSG, they need to be associated with a project. Set the environment variable OSG_PROJECT_NAME to the project (for example USC_Deelman) you are associated with.   

### Latex Notes for Plots Generation

Generation of the figures requires a LaTeX installation on the machine where `make_pycbc_hist.sh` is run (for example a [texlive](https://www.tug.org/texlive/) install). In addition, the [Arev Sans](https://ctan.org/tex-archive/fonts/arev/?lang=en) fonts need to be installed. To install these on a Linux texlive installation, download http://mirrors.ctan.org/fonts/arev.zip and unzip this file. Copy the `tex` and `fonts` directories to the appropriate place for your texlive install, e.g.
```
cp -R arev/tex/latex/arev /usr/share/texlive/texmf-local/texmf-compat/tex/latex
cp -R arev/fonts /usr/share/texlive/texmf-local/texmf-compat/fonts
```
Run the commands
```
mktexlsr
updmap-sys --force --enable Map=arev.map
mktexlsr
```
to install the extra Arev Sans fonts. The install the `texlive-mathdesign` fonts by running the command
```
yum install texlive-mathdesign
```
or similar for your installation.

