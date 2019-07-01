---
title: "RStudio deployment for fun and profit"
teaching: 0
exercises: 20
questions:
objectives:
- Run an R workflow through both the terminal and RStudio using containers
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
