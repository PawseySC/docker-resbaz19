---
title: "RStudio deployment for fun and profit"
teaching: 0
exercises: 30
questions:
objectives:
- Run an R workflow through both the terminal and RStudio using containers
- Deploy a customised RStudio container for bioinformatics
keypoints:
- Containers are great way to manage R workflows.  You likely still want to have a local installation of R/Rstudio for some testing, but if you have set workflows, you can use containers to manage them.  You can also provide Rstudio servers for collaborators
---

### RStudio example ###

R is a popular language in several domains of science, particularly because of its statistical packages.  It often requires installing a large number of dependencies, and installing these on an HPC system can be tedious.

Instead we can use an R container to simplify the process.


### Rocker ###

The group [Rocker](https://hub.docker.com/r/rocker) has published a large number of R images we can use, including an Rstudio image.  To begin, we'll pull a Tidyverse container image (contains R, RStudio, data science packages):

```
$ docker pull rocker/tidyverse:3.5
```
{: .bash}


### Running a scripted R workflow on the shell ###

Let us `cd` to the `06_rstudio` demo directory:

```
$ cd <top-level>/demos/06_rstudio
```
{: .bash}

We are going to use an example example that has been derived from the workshop [Programming with R](http://swcarpentry.github.io/r-novice-inflammation/) by the Software Carpentry.
In particular, their [Episode 5](http://swcarpentry.github.io/r-novice-inflammation/05-cmdline/index.html) is the source for the dataset and the R script file; the latter has been adapted for this workshop.


Now, let us run the R script using the R container we pulled; we're going to compute average values in this example:

```
$ docker run -v `pwd`:/data -w /data rocker/tidyverse:3.5 Rscript readings-density.R --mean inflammation-density.png data/inflammation-*.csv
```
{: .bash}

```
5.45
5.425
6.1
[..]
6.875
6.025
6.9
Saving 7 x 7 in image
```
{: .output}

The analysis outputs a plot in a picture file, `inflammation-density.png`, that can be opened and visualised.


### Using an RStudio web server to run the workflow ###

Let us setup an RStudio web server using Docker, by using the following command:

```
$ docker run --rm -d -p 80:8787 --name=rstudio -v `pwd`:/home/rstudio -e PASSWORD=<Pick your password> rocker/tidyverse:3.5
```
{: .bash}

Here we're opening up the container port `8787` and mapping it to the host port `80` so we can access the Rtudio server remotely. Note you need to store a password in a variable; it will be required below for the web login. Also, we are mapping the current directory into `home/rstudio` in the container; this is the default RStudio working directory in Rocker containers.

You just need to open a web browser and point it to `localhost` if you are running Docker on your machine, or `<Your VM's IP Address>` if you are running on a cloud service.

You should see a prompt for credentials, with user defaulting to `rstudio`, and password..

Now you can run the same analysis from the RStudio console:

```
> source("readings-density.R")
```
{: .r}

This time the output plot is saved in the file `interactive.png`.

Once you're done, stop the container with:

```
$ docker stop rstudio
```
{: .bash}


### Customised RStudio images ###

The above example only provides a bare-bones RStudio image, but now we want to actually use some R packages.  The following example is based on a bioinformatics workshop at [OzSingleCell2018](https://github.com/IMB-Computational-Genomics-Lab/SingleCells2018Workshop).  We'll use their data for our Docker/Rstudio example.

To begin, let's `cd` to the `06_rstudio_bio` demo directory, where a trimmed down repo with their data has been created for this tutorial:

```
$ cd <top-level>/demos/06_rstudio_bio
```
{: .bash}

For this example, we'll use an RStudio image thas has already been built.  R images can take a while to build sometimes, depending on the number of packages and dependencies you're installing.  The Dockerfile used here is included, and we'll briefly comment through it.
 
```
FROM rocker/tidyverse:3.5

RUN apt-get update -qq && apt-get -y --no-install-recommends install \
      autoconf \
      automake \
      g++ \
      gcc \
      gfortran \
      make \
      && apt-get clean all \
      && rm -rf /var/lib/apt/lists/*

RUN mkdir -p $HOME/.R
COPY Makevars /root/.R/Makevars

RUN Rscript -e "library('devtools')" \
      -e "install_github('Rdatatable/data.table', build_vignettes=FALSE)" \
      -e "install.packages('reshape2')" \
      -e "install.packages('fields')" \
      -e "install.packages('ggbeeswarm')" \
      -e "install.packages('gridExtra')" \
      -e "install.packages('dynamicTreeCut')" \
      -e "install.packages('DEoptimR')" \
      -e "install.packages('http://cran.r-project.org/src/contrib/Archive/robustbase/robustbase_0.90-2.tar.gz', repos=NULL, type='source')" \
      -e "install.packages('dendextend')" \
      -e "install.packages('RColorBrewer')" \
      -e "install.packages('locfit')" \
      -e "install.packages('KernSmooth')" \
      -e "install.packages('BiocManager')" \
      -e "source('http://bioconductor.org/biocLite.R')" \
      -e "biocLite('Biobase')" \
      -e "biocLite('BioGenerics')" \
      -e "biocLite('BiocParallel')" \
      -e "biocLite('SingleCellExperiment')" \
      -e "biocLite('GenomeInfoDb')" \
      -e "biocLite('GenomeInfgoDbData')" \
      -e "biocLite('DESeq')" \
      -e "biocLite('DESeq2')" \
      -e "BiocManager::install(c('scater', 'scran'))" \
      -e "library('devtools')" \
      -e "install_github('IMB-Computational-Genomics-Lab/ascend', ref = 'devel')" \
      && rm -rf /tmp/downloaded_packages
```
{: .source}

The first line, `FROM`, specifies a base image to use.  We could build up a full R image from scratch, but why waste the time.  We can use Rocker's pre-built image to start with and simplify our lives.

`RUN apt-get update` is installing some packages we'll need via Ubuntu's package manager.  Really all we're installing here are compilers.

The next section adds some flags and options we want to use when building R packages, by copying a file from the build context, `Makvevars`.

The last section is the main R package installation section.  Here we run several different installation methods:

* `install.packages()` is R's standard method for installing packages from CRAN.  We also install the `robustbase` package from source here.
* `biocLite()` is [Bioconductor's](https://bioconductor.org) method for installing packages
* `BiocManager` is a [CRAN package](https://cran.r-project.org/package=BiocManager) for installing bioinformatics software
* `install_github()` is method for installing R packages from GitHub.

We'll skip building this image for now, and just pull and use a prebuilt image:

```
$ docker pull bskjerven/oz_sc:latest
```
{: .bash}

Now let us setup an RStudio server based on it:

```
$ docker run --rm -d -p 80:8787 --name=rstudio -v `pwd`:/home/rstudio -e PASSWORD=<Pick your password> bskjerven/oz_sc:latest
```
{: .bash}

Shortly after this starts, open a web browser and go to `localhost` if you are running Docker on your machine, or `<Your VM's IP Address>` if you are running on a cloud service. You should see an Rstudio login, and we've set the username to `rstudio` and password..

Once logged in, you type (note this is the R shell):

```
> source('data/SC_script.r')
```
{: .r}

to run the tutorial (it may take a few minutes).  We can refer to the [OzSingleCell2018](https://github.com/IMB-Computational-Genomics-Lab/SingleCells2018Workshop) repo for details on each step.

To stop your Rstudio image, simply type:

```
$ docker stop rstudio
```
{: .bash}
