# Guide for contributors

This repo is intended to serve PReSto contributors. The parameter input standards for presto reconstructions (presto_input_standards) and an example (Holocene_DA_parameters.yml) live here. There are also containerization instructions (containerization_instructions.md) and you will find links to the repositories for two example reconstructions, including Dockerfiles.

## Step 1 - Formatting your input paramters
all parameters for your reconstruction will need to be applied by a single yaml file

the yaml file will need to conform to the standards outlined in presto_input_standards.md

a completed example can be found in Holocene_DA_paramters.yml

## Step 2 - Formatting your outputs
need to complete this section

## Step 3 - Containerizing your code
containerizing you code with Docker removes any concern of changing dependencies and variability across operating systems

* if you do not have Docker on your local machine, you will first need to download and install it: [Docker Desktop](https://www.docker.com/products/docker-desktop/)
* gather all files necessary for your reconstruction in one place (remember, data should be pulled from a repository when the container is run, so it will not be included here)
* write a Dockerfile
*   see an example in R here:
*   see an example in Python here:
* test your new container by building and running it locally
* after debugging, push this directory to GitHub  
