---
title: "Cleaning up Docker"
teaching: 10
exercises: 5
questions:
objectives:
- Learn how to remove containers and images from your machine when you no longer need them
keypoints:
- "Cleaning up containers and images is a two-step process"
- "Remove stopped containers with `docker rm`"
- "Delete unnecessary images with `docker rmi`"
- "`docker run --rm` allows to automatically remove containers at completion"
---

### Cleaning up ###

Eventually, you may want to clean out unnecessary images and the cache of containers, to reclaim space or just keep things tidy. Cleaning up involves two steps, removing the containers that you've run first, then removing the images themselves.

To remove the containers, including those that have exited and are still in the cache, use `docker ps --all` to get the IDs, then `docker rm` followed by the container ID(s) or the container name(s):

```
$ docker ps --all
```
{: .bash}

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                        PORTS               NAMES
48a2dca14407        nginx               "nginx -g 'daemon off"   10 minutes ago      Exited (0) 4 minutes ago                          pensive_booth
1bf15ca4ba89        nginx               "nginx -g 'daemon off"   20 minutes ago      Exited (0) 16 minutes ago                         amazing_hugle
97b1e86df1f1        ubuntu              "/bin/bash"              37 minutes ago      Exited (0) 37 minutes ago                         backstabbing_brown
1688f55c3418        ubuntu              "/bin/bash"              37 minutes ago      Exited (127) 20 minutes ago                       ecstatic_hugle
c69d6f8d89bd        ubuntu              "/bin/bash"              41 minutes ago      Exited (0) 41 minutes ago                         sad_keller
960588723c36        ubuntu              "/bin/echo 'hello wor"   45 minutes ago      Exited (0) 45 minutes ago                         suspicious_mestorf
ffbb0f60bda6        ubuntu              "/bin/echo 'hello wor"   51 minutes ago      Exited (0) 51 minutes ago                         mad_booth
```
{: .output}

This is how cleaning by ID looks like:

```
$ docker rm ffbb0f60bda6 960588723c36
```
{: .bash}

```
ffbb0f60bda6
960588723c36
```
{: .output}

And this is cleaning by name:

```
$ docker rm sad_keller ecstatic_hugle
```
{: .bash}

```
sad_keller
ecstatic_hugle
```
{: .output}

Let's check the list of containers again, it will be shorter:

```
$ docker ps --all
```
{: .bash}

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS               NAMES
48a2dca14407        nginx               "nginx -g 'daemon off"   23 minutes ago      Exited (0) 17 minutes ago                       pensive_booth
1bf15ca4ba89        nginx               "nginx -g 'daemon off"   33 minutes ago      Exited (0) 29 minutes ago                       amazing_hugle
97b1e86df1f1        ubuntu              "/bin/bash"              50 minutes ago      Exited (0) 50 minutes ago                       backstabbing_brown
```
{: .output}

You can construct your own one-liner to clean up everything, if you wish. Docker will refuse to delete something that's actively in use, so you won't screw things up too badly this way.

```
$ docker rm `docker ps --all -q`
```
{: .bash}

```
48a2dca14407
1bf15ca4ba89
97b1e86df1f1
```
{: .output}

Now that we've removed the containers, let's clean up the images they came from. This uses the `docker images` and `docker rmi` commands, in a similar manner, to list images:

```
$ docker images
```
{: .bash}

```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              4ca3a192ff2a        22 hours ago        128.2 MB
nginx               latest              abf312888d13        2 days ago          181.5 MB
```
{: .output}

... and to remove specific image(s):

```
$ docker rmi ubuntu
```
{: .bash}

```
Untagged: ubuntu:latest
Untagged: ubuntu@sha256:3b64c309deae7ab0f7dbdd42b6b326261ccd6261da5d88396439353162703fb5
Deleted: sha256:4ca3a192ff2a5b7e225e81dc006b6379c10776ed3619757a65608cb72de0a7f6
Deleted: sha256:2c2e0ef08d4988122fddadfe7e84d7b0aae246e6fa805bb6380892a325bc5216
Deleted: sha256:918fe8ae141d41d7430eb887e57a21d76fb0181317ec4a68f7abbd17caab034a
Deleted: sha256:00f434fa2fa1a0558f8740af28aef3d6ee546a8758c0a37dddee3f65b5290e7a
Deleted: sha256:f9545ee77b4b95c887dbebc6d08325be354274423112b0b66f7288a2cf7905cb
Deleted: sha256:d29d52f94ad5aa750bd76d24effaf6aeec487d530e262597763e56065a06ee67
```
{: .output}

You need to be sure to stop and remove a container before removing its image.  If not, you'll see an error about child images:

```
$ docker ps --all
```
{: .bash}

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                        PORTS                  NAMES
bf0a1726909a        nginx               "nginx -g 'daemon of…"   18 minutes ago      Exited (0) 16 minutes ago                            eloquent_raman
```
{: .output}

```
$ docker rmi nginx
```
{: .bash}

```
Error response from daemon: conflict: unable to remove repository reference "nginx" (must force) - container bf0a1726909a is using its referenced image 71c43202b8ac
```
{: .error}


### Self-cleaning when running a container ###

It's possible to run a Docker container with an additional flag, `--rm`, so that the container is automatically removed from the local cache once execution terminates. This way no later cache clean up is required.

Let's see it with an example. We'll clean up the cache first,

```
$ docker rm `docker ps --all -q`
```
{: .bash}

Then we'll check the cache is empty:

```
$ docker ps -a
```
{: .bash}

```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
{: .output}

Now let's run a container with the `--rm` flag:

```
$ docker run --rm ubuntu echo "hello world"
```
{: .bash}

And finally we'll check that the cache is still empty, as a consequence of having used the `--rm` flag:

```
$ docker ps -a
```
{: .bash}

```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
{: .output}

Note that while the `--rm` flag cleans the exited container from cache, it preserves the image locally, so that further runs are allowed, until `docker rmi` is used.


### Dangling & Unused Images ###

After building and pulling images you may eventually find that your list of images includes with the name `<none>:<none>`:

```
$ docker images
```
{: .bash}

```
REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
<none>                         <none>              bc47408f7de6        14 hours ago        141MB
<none>                         <none>              9cbb4d4f03e4        14 hours ago        135MB
pytorch-jupyter                latest              1fc491b0444e        20 hours ago        5.69GB
resbaz-plot-ex                 latest              2bf0de9138f1        21 hours ago        5.72GB
rocker/tidyverse               latest              1692d80ba11b        33 hours ago        2.1GB
```
{: .output}

These are what are known as **dangling images**.  These are produced as a result of `docker pull` and `docker build`.  If you rebuild an image, but don't rename it, the old image is left *dangling*.
Dangling images are not tagged and not referenced by anohter container.  One issue with these is that they can take up disk space, so periodically you may need to remove them.  Rather than trying to keep track of all the image IDs and names, we use a single command to query the daingling images:

```
$ docker images -q -f dangling=true
```
{: .bash}

and we can easily remove all dangling images with:

```
$ docker image prune
```
{: .bash}

You can also build up **unused images**.  When you pull an image it's pulled layer by layer.  Only the final layer is given a name like `ubuntu:latest`.  The other images are unused, but technically
associated with another image.  The good news is that unused images don't take up disk space.  If you want to remove all unused images (in addition to all dangling images), we use the `-a` flag:

```
$ docker image prune -a
```
{: .bash}

We won't cover it in this course, but Docker can also be used to set up virtual networks and volumes.  If you wish to prune ***everything*** (stopped containers, unused/dangling images, build cache, networks, & volumes) 
you can use the following command:

```
$ docker system prune --volumes
```
{: .bash}

**WARNING:** This will remove everything, and may result in data loss


> ## Clean up Miniconda containers from previous episode ##
> 
> First, get the list of all stopped containers you have run so far.
> 
> > ## Solution ##
> > 
> > ```
> > $ docker ps -a
> > ```
> > {: .bash}
> > 
> > Sample output:
> > 
> > ```
> > CONTAINER ID        IMAGE                           COMMAND                  CREATED              STATUS                          PORTS               NAMES
> > ed6ccf413d3e        continuumio/miniconda3:4.5.12   "/usr/bin/tini -- py…"   22 seconds ago       Exited (0) 20 seconds ago                           quizzical_darwin
> > 6f872cae5a24        continuumio/miniconda3:4.5.12   "/usr/bin/tini -- py…"   27 seconds ago       Exited (0) 26 seconds ago                           infallible_swanson
> > 4386dd7b1e48        ubuntu                          "/bin/bash"              42 seconds ago       Exited (0) 37 seconds ago                           youthful_ardinghelli
> > 5fbb957bd7f0        ubuntu                          "/bin/echo 'hello wo…"   50 seconds ago       Exited (0) 49 seconds ago                           mystifying_poincare
> > f06c68c27ab2        ubuntu:17.04                    "cat /etc/os-release"    55 seconds ago       Exited (0) 54 seconds ago                           affectionate_booth
> > b236634fcd6b        ubuntu                          "cat /etc/os-release"    About a minute ago   Exited (0) About a minute ago                       admiring_mestorf
> > ```
> > {: .output}
> {: .solution}
> 
> Then, identify the stopped `miniconda3` containers you ran in the previous episode. You can use either the IDs or NAMES column. 
> 
> Hint: you can identify them as they will correspond to the image `continuumio/miniconda3:4.5.12`.
> 
> > ## Sample solution ##
> > 
> > IDs: `ed6ccf413d3e` and `6f872cae5a24`
> > 
> > Names: `quizzical_darwin` and `infallible_swanson`
> {: .solution}
> 
> Finally, remove the `miniconda3` stopped containers.
> 
> > ## Solution ##
> > 
> > ```
> > $ docker rm <list of IDs>
> > ```
> > {: .bash}
> > 
> > or 
> > 
> > ```
> > $ docker rm <list of names>
> > ```
> > {: .bash}
> > 
> > Sample commands:
> > ```
> > $ docker rm ed6ccf413d3e 6f872cae5a24
> > ```
> > {: .bash}
> > 
> > ```
> > ed6ccf413d3e
> > 6f872cae5a24
> > ```
> > {: .output}
> > 
> > or
> > 
> > ```
> > $ docker rm quizzical_darwin infallible_swanson
> > ```
> > {: .bash}
> > 
> > ```
> > quizzical_darwin
> > infallible_swanson
> > ```
> > {: .output}
> {: .solution}
> 
> Let us NOT remove the `miniconda3` image for now.
{: .challenge}


> ## Best practices ##
> 
> * Use `docker run` with the `--rm` flag when you know you won't want to re-start a container
> * If you use containers heavily, clean up the images from time to time (`docker image prune`)
{: .callout}
