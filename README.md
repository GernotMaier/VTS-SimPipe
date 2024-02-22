# VERITAS Simulation Pipeline

[![DOI](https://zenodo.org/badge/738007615.svg)](https://zenodo.org/doi/10.5281/zenodo.10541349)
[![License](https://img.shields.io/badge/License-BSD_3--Clause-blue.svg)](https://opensource.org/licenses/BSD-3-Clause)
[![build-images](https://github.com/VERITAS-Observatory/VTS-SimPipe/actions/workflows/build-images.yml/badge.svg)](https://github.com/VERITAS-Observatory/VTS-SimPipe/actions/workflows/build-images.yml)

[VERITAS](https://veritas.sao.arizona.edu/) is a ground-based gamma-ray observatory located at the Fred Lawrence Whipple Observatory in southern Arizona, USA.
It explores the gamma-ray sky in the energy range from 100 GeV to 30 TeV.

The VERITAS simulation pipeline **VTS-SimPipe** (this repository) is a set of scripts to run the simulation of air showers and the Cherenkov light emission, propagation, and detection in the VERITAS telescope system.
This repository stores also a copy of the simulation model for the atmospheric conditions, telescope structure, and camera and readout of VERITAS.

A detailed documentation on the derivation of these parameters can be found on the VERITAS internal pages:

- [VERITAS Simulations (private wiki page)](https://veritas.sao.arizona.edu/wiki/Corsika) (quite outdated)
- [VERITAS CARE (private wiki page)](https://veritas.sao.arizona.edu/wiki/index.php/CARE)

This work is built on a large effort from many people, especially:

- the [CORSIKA](https://web.ikp.kit.edu/corsika/) team
- Charlie Duke for the [GrOptics](https://github.com/groptics/GrOptics/tree/master) package
- Nepomuk Otte for the [CARE](https://github.com/nepomukotte/CARE) package
- Raul Prado for an initial pipeline implementation for DESY (see [here](https://github.com/RaulRPrado/MC-DESY/tree/master))
- Tony Lin for a Docker implementation of the pipeline (see [here](https://github.com/VERITAS-Observatory/Build_SimDockerImage/tree/master))
- Luisa Arrabito and Orel Gueta on providing the optimized CORSIKA code and help with compilation

## Quick startup

The following is all what you need to know to install and run the simulation pipeline.
No compilation of any of the package is required.

```bash
# clone repository
git clone https://github.com/VERITAS-Observatory/VTS-SimPipe.git
cd VTS-SimPipe
# prepare log files and directories
cp env_setup_template.sh env_setup.sh
# edit env_setup.sh to your needs
# pull all container images from the registry
cd scripts && ./pull.sh
# prepare your configuration (e.g. zenith angle, number of events, etc.)
# see example in config/config_ATM61_template.dat
# prepare production
cd scripts
./prepare_all_production_steps.sh \
   ../config/config_ATM61_template.dat
   ../config/CORSIKA/input_template.dat
# on DESY: log into the DAG submission node
./prepare_DAG_jobs.sh ../config/config_ATM61_template.dat
./submit_DAG_jobs.sh ../config/config_ATM61_template.dat <directory with DAG files>
# otherwise: submit jobs to HT Condor - for each step (CORSIKA, GROPTICS, CARE)
./submit_jobs_to_htcondor.sh <directory with condor files / submission scripts> submit
# now wait....for jobs to finish
# merge vbf files
./prepare_production.sh ../config/config_ATM61_template.dat
./submit_jobs_to_htcondor.sh <directory with condor files for mergeVBF> submit
# that's it
```

## Software packages

Following software packages are used by the simulation pipeline and installed in the docker images:

- [CORSIKA](https://web.ikp.kit.edu/corsika/) for air shower and Cherenkov photon generation
- [corsikaIOreader](https://github.com/GernotMaier/corsikaIOreader/) for file format conversion and Cherenkov photon absorption and scattering.
- [GrOptics](https://github.com/groptics/GrOptics/tree/master) for the optical ray tracing (uses [ROBAST](https://github.com/ROBAST/ROBAST)).
- [CARE](https://github.com/nepomukotte/CARE) for the camera simulation.
- [VBF](https://github.com/VERITAS-Observatory/VBF) for the VBF file format (internal software of the VERITAS collaboration).
- [ROOT](https://root.cern.ch/) used by GrOptics and CARE.
- [Eventdisplay](https://github.com/VERITAS-Observatory/EventDisplay_v4) to merge several VBF files.

For software versions, see the [docker files](docker/Dockerfile) and the [release notes](https://github.com/VERITAS-Observatory/VTS-SimPipe/releases).

## Installation

The simulation pipeline is configured to run in Docker/Apptainer containers.
Images can be downloaded from the package registry of this repository.

### CORSIKA containers

Requires the tar package with the CORSIKA tar software to be available in the main directory of `VTSSimPipe`.
Note that the CI on github will build three different containers for CORSIKA:

1. [vts-simpipe-corsika](https://github.com/VERITAS-Observatory/VTS-SimPipe/pkgs/container/vtsimpipe-corsika) based on [docker/Dockerfile-corsika](docker/Dockerfile-corsika) with the standard CORSIKA software (as used in VERITAS for productions in the past)
2. [vts-simpipe-corsika-noopt](https://github.com/VERITAS-Observatory/VTS-SimPipe/pkgs/container/vtsimpipe-corsika-noopt) based on [docker/Dockerfile-corsika-noopt](docker/Dockerfile-corsika-noopt) using CORSIKA 7.7500 with minor updates to the Bernlohr package (this is the package used for the generation and propagation of Cherenkov photons).
3. [vts-simpipe-corsika-ax2](https://github.com/VERITAS-Observatory/VTS-SimPipe/pkgs/container/vtsimpipe-corsika-ax2) based on [docker/Dockerfile-corsika-ax2](docker/Dockerfile-corsika-ax2) using CORSIKA 7.7500 with minor updates to the Bernlohr package (this is the package used for the generation and propagation of Cherenkov photons). A patch is applied to the Cherenkov photon code to allow to use vector instructions and improve runtime performance.

To build the CORSIKA container (similar for all):

```bash
docker build -f ./docker/Dockerfile-corsika -t vts-simpipe-corsika .
```

For testing, run the container with:

```bash
docker run --rm -it -v "$(pwd):/workdir/external" vts-simpipe-corsika bash
```

The CORSIKA configuration is generated "by hand" using the `coconut` tools with the following settings:

```text
options:   VOLUMEDET TIMEAUTO URQMD QGSJETII
selection: BERNLOHRDIR SLANT CERENKOV IACT IACTDIR ATMEXT
```

The file [docker/corsika-config.h](docker/corsika-config.h) contains the configuration file for CORSIKA and is used for the compilation ([docker/corsika-config-ax2.h](docker/corsika-config-ax2.h) for the `vts-simpipe-corsika-noopt` and `vts-simpipe-corsika-axi2` options).

### GrOptics and corsikaIOreader containers

GrOptics requires C++11 for compilation, therefore the `root:6.24.06-centos7` is used as base image.

```bash
docker build -f ./docker/Dockerfile-groptics -t vts-simpipe-groptics .
docker run --rm -it -v "$(pwd):/workdir/external" vts-simpipe-groptics bash
```

Note that at this point a fork of GrOptics is used, fixing an issue to allow compiling on cent os7 (see [fork here](https://github.com/GernotMaier/GrOptics)).

### CARE containers

The same base image for CARE is used as for GrOptics, including VBF and ROOT.

```bash
docker build -f ./docker/Dockerfile-care -t vts-simpipe-care .
docker run --rm -it -v "$(pwd):/workdir/external" vts-simpipe-care bash
```

### mergeVBF containers

The tiny tool mergeVBF is actual part of the Eventdisplay software.
The same base image for mergeVBF is used as for CARE, including VBF and ROOT.
This container also includes an installation of [zstd](https://github.com/facebook/zstd), which is used to compress the VBF files.

```bash
docker build -f ./docker/Dockerfile-mergevbf -t vts-simpipe-mergevbf .
docker run --rm -it -v "$(pwd):/workdir/external" vts-simpipe-mergevbf bash
```

## Environmental Variables

Environmental variables are used to configure the simulation pipelines, especially output directories and executables.
Copy the file [env_setup_template.sh](env_setup_template.sh) to `env_setup.sh` and modify the variables for your local needs.

Directories:

- `VTSSIMPIPE_LOG_DIR`: directory for log files, run scripts, and job submission files
- `VTSSIMPIPE_DATA_DIR`: directory for simulation output files

## Configuration

### CORSIKA air-shower simulations

The CORSIKA air-shower simulations require as environmental variables:

- `VTSSIMPIPE_IMAGE` pointing to the Docker image to be used for the simulations
- `VTSSIMPIPE_CORSIKA_EXE` describes the type of container platform (docker or apptainer) or points directly to the CORSIKA executable (in cases no containers are used)

The script to run the CORSIKA simulations is [scripts/run_corsika.sh](scripts/run_corsika.sh), configuration parameters are
defined as in the template [config/CORSIKA/config_template.dat](config/CORSIKA/config_template.dat).
Note that no changes are expected to be necessary for the input steering template [config/CORISKA/input_template.dat](config/CORISKA/input_template.dat).

The configuration script [scripts/config_corsika.sh](scripts/config_corsika.sh) takes into account the zenith-angle dependent changes of energy thresholds and core scatter areas. It is recommended to cross check these values.

Usage of CORSIKA run script:

```bash
./prepare_production_corsika.sh ../config/CORSIKA/config_template.dat ../config/CORSIKA/input_template.dat TRUE
```

The last parameter indicates to pull the container image from the registry.
For using `apptainers`, the tables required by the interaction models are copied in this step also to the `VTSSIMPIPE_CORSIKA_DIR` (as apptainers containers are not writeable and QGSJet requires write access to the tables).

### corsikaIOreader for absorption and scattering

After the CORSIKA simulations, the Cherenkov photons are absorbed and scattered using the `corsikaIOreader` package.
The optical depth as function of altitude and wavelength is stored in files for summer and winter in [config/ATMOSPHERE](config/ATMOSPHERE).
For details on the derivation of these tables, see the [internal VERITAS wiki](https://veritas.sao.arizona.edu/wiki/Atmosphere) (plus pages linked from there).

### GrOptics optical ray tracing

see configuration files in [config/TELESCOPE_MODEL](config/TELESCOPE_MODEL).

### CARE camera simulation

see configuration files in [config/TELESCOPE_MODEL](config/TELESCOPE_MODEL).

## Processing Scripts

Processing scripts are prepared for HT Condor systems.

- prepare run scripts with [scripts/prepare_production.sh](scripts/prepare_production.sh) (see `*.sh` files in the `VTSSIMPIPE_LOG_DIR` directory).
- job submission for the HT Condor system is done with [scripts/submit_jobs_to_htcondor.sh](scripts/submit_jobs_to_htcondor.sh).
- production scripts for all steps can be prepared with [scripts/prepare_all_production_steps.sh](scripts/prepare_all_production_steps.sh).
- DAG submission is done with [scripts/submit_DAG_jobs.sh](scripts/submit_DAG_jobs.sh).
- note that the `MERGEVBF` step is not included in the DAG submission, as it combines all runs of a production. Run this as a final step at the ned of the production.

Note that configuration and output directories are fine tuned for this setup.
The preparation of the all temporary submission scripts is not very efficient in both time and number of files written (could be significantly improved).

## Using Apptainers

[Apptainer](https://apptainer.org/) is a container platform often used on HPC systems (e.g., on the DESY computing cluster).
The simulation scripts are configured to use Apptainers with the correct parameters.

Note:

- recommend to set `$APPTAINER_CACHEDIR` to a reasonable directory with sufficient disk space, as cluster jobs will use this directory to store the container images.
- set `$VTSSIMPIPE_CONTAINER_DIR` to the directory where the container images are stored. This should be a "fast" disk, as each job will access the image files.

## Submitting jobs

The job submission scripts are written for HT Condor systems (as configured at DESY).
The `prepare_production.sh` product scripts generates the condor files and submission scripts in the `VTSSIMPIPE_LOG_DIR` directory.

To submit jobs, use the `submit_jobs_to_htcondor.sh` script:

```bash
./submit_jobs_to_htcondor.sh <directory with condor files / submission scripts> submit
````

This will submit all jobs in the directory to the HT Condor system.

For efficiency reason, it is recommended to submit jobs using the DAG submission system (see [HTCondor DAGMan](https://htcondor.readthedocs.io/en/latest/automated-workflows/index.html)). This allows to run CORSIKA, followed by GrOptics, CARE, and the merging of the VBF files in a single job submission.

To generate DAG files for the job submission:

```bash
./prepare_DAG_jobs.sh <config file>
```

This will generate the DAG files in the `VTSSIMPIPE_LOG_DIR/DAG` directory.

To submit DAG jobs:

```bash
./submit_DAG_jobs.sh <config file> <directory with DAG files>
```

Note that on DESY DAG jobs need to be submitted from a special node.

## Notes on simulation model and configuration parameters

- v0.1.0: initial version with configuration parameters as used at the DESY production site
- v0.2.0: optimized configuration parameters for the CORSIKA simulations

Configuration:

- CORSIKA: NSHOW 2000 (roughly 10h per job; 2 GB)
- used 5x (core scatter) -> 10,000 events for GrOptics / CARE per file
- total number of event stdHV: `1.e7`
- total number of events redHV: `1.5e+07`
- ze: `0 00 20 30 35 40 45 50 55 60 65`
- az: [0, 360] deg
- wobble offsets: `0.0 0.25 0.5 0.75 1.0 1.25 1.5 1.75 2.0`
- NSB std: `50 75 100 130 160 200 250 300 350 400 450 600`
- NSB redHV: `150 300 450 600 750 900`

### CORSIKA parameters

#### 2017 production

```Text
Monte Carlo run header
======================
run number: 1
code version: shower prog 1 (7000), detector prog 0 (0), convert 1440
date: 170529    20210301
number of showers: 0 (each shower used 5 times)
Primary 1
Energy range: [0.03, 200] TeV, powerlaw index -1.5
Core scattering: 750    0 [m] (circular)
Azimuth range: [0, 360]
Zenith range: [20, 20]
Viewcone: [0, 0] (0)
Observatory height 1270 [m]
B-Field: 48.0231 microT (58.3487,0.1815)
Atmospheric model: 61
Cherenkov photon wavelength range: [200, 700]
CORSIKA interaction detail: lowE 0, highE 0, interaction models: lowE 2, highE 3, transition energy 100 GeV
CORSIKA iact options: 62907
CERENKOV 1
IACT 1
CEFFIC 0
ATMEXT 1
ATMEXT with refraction 1
VOLUMEDET 1
CURVED 0
SLANT 1
```

#### 2024 production

- changed wavelength range from [200, 700] to [300, 600] nm

### corsikaIOreader parameters

- apply pre-efficiency cut of 50%
