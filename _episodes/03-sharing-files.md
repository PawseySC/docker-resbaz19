---
title: "Sharing files with the host with Docker"
teaching: 20
exercises: 0
questions:
objectives:
- Learn how to mount host directories in a container
- Learn how to set custom user and group IDs in a container

keypoints:
---

### Directory and file defaults in Docker ###

 We are going to use the bash syntax `bash -c '<..> ; <..>'` to run multiple commands at the same time within the same container.
 Try and run the following:

```
> docker run ubuntu bash -c 'pwd ; ls -l'
/
total 64
drwxr-xr-x   2 root root 4096 Nov 12 20:56 bin
drwxr-xr-x   2 root root 4096 Apr 24  2018 boot
drwxr-xr-x   5 root root  340 Dec 19 08:01 dev
drwxr-xr-x   1 root root 4096 Dec 19 08:01 etc
drwxr-xr-x   2 root root 4096 Apr 24  2018 home
drwxr-xr-x   8 root root 4096 Nov 12 20:54 lib
drwxr-xr-x   2 root root 4096 Nov 12 20:55 lib64
drwxr-xr-x   2 root root 4096 Nov 12 20:54 media
drwxr-xr-x   2 root root 4096 Nov 12 20:54 mnt
drwxr-xr-x   2 root root 4096 Nov 12 20:54 opt
dr-xr-xr-x 125 root root    0 Dec 19 08:01 proc
drwx------   2 root root 4096 Nov 12 20:56 root
drwxr-xr-x   1 root root 4096 Nov 19 21:20 run
drwxr-xr-x   1 root root 4096 Nov 19 21:20 sbin
drwxr-xr-x   2 root root 4096 Nov 12 20:54 srv
dr-xr-xr-x  13 root root    0 Dec 14 13:27 sys
drwxrwxrwt   2 root root 4096 Nov 12 20:56 tmp
drwxr-xr-x   1 root root 4096 Nov 12 20:54 usr
drwxr-xr-x   1 root root 4096 Nov 12 20:56 var
```

You are in the root `/` directory of the container, and if you compare the listing of directories with what you get in the host (type `ls -l /` for this), you will notice the two are different; even directories with the same name will have in general different timestamps, suggesting they are in fact distinct directories.

Now try and create an empty file and then see who is the owner:

```
> docker run ubuntu bash -c 'touch empty-file ; ls -l empty-file'
-rw-r--r-- 1 root root 0 Dec 19 08:06 empty-file
```
The file is owned by the root user!

What we have just seen is a consequence of some Docker defaults:

- a container hasn't got any access to directories in the host filesystem (i.e. directories in the computer where you're running the container from)
- as by default a container is run as root, any created file is owned by the group user.

Let's see how these defaults can be overcome if you want to read/write on host directories effectively.


### Accessing host directories ###

Docker has the ability to mount host directories into a container.  This allows you to add data to your container, as well as specify output directories you can use to store data after a container ends.  This is extremely useful as it's a bad idea to package up your containers with lots of data; it increases the size of the containers and makes them less portable (what if someone else wants to run the same container with different data?).

![Docker Volumes]({{ page.root }}/fig/docker-volume.png)

The docker daemon has a parameter called volume (`-v` or `--volume`), which we'll use to specify directories to be mounted.

The format is `-v /host/path:/container/path`.  Docker will create the directory inside the container if required.  Be aware the behaviour is different if you use absolute or relative paths, we use absolute paths here.

As an example, let us run the following:

```
> docker run -v `pwd`:/data ubuntu ls -l /data
```

Here we are using ``` `pwd` ``` as a shortcut for the current working directory. As a result of using the mapping option `-v`, the `ls` command run inside the container will display the content of the current directory in the host.

The `-v` flag maps host directories in the container, allowing to read/write within them. Let us use a container to create a file in a mapped directory:

```
> docker run -v `pwd`:/data ubuntu touch /data/container1
```

Now, let us look for that file in the host:

```
> ls -l container1 
-rw-r--r-- 1 root root 0 Dec 19 08:16 container1
```

The file created in the container is actually available from the host, as a consequence of volume mapping.

Finally, Docker has a flag to change working directory in the container, to avoid using full paths, `-w` or `--workdir`; for instance let us use it to change dir to the mapped host directory:

```
docker run -v `pwd`:/data -w /data ubuntu touch container2
```

```
> ls -l container2
-rw-r--r-- 1 root root 0 Dec 19 08:19 container2
```

This can be useful to make your workflow uniform, as different container providers may have different default working directorries.


### More on Volumes ###

Docker has several ways to mount data into containers:

* **-v (or --volume)** 
* **--mount** 

There are some subtle differences, but here are the key points:

* Volume mounts (`-v`) will create a new directory in the container, and mount your external directory there, even if the container directory doesn't exist (`--mount` will generate an error if the internal directory isn't present)
* `--mount` is generally more performant, but requires you to be explicit in your mount command, and requires that your host machine has a specific directory structure.

In general, start with **-v**.


### Matching user permissions with the host ###

So far we have seen that files created by the container belong to the root user. This can be annoying, in that the host user might then have limitations in editing/deleting those files. 
Docker has an option, `-u` or `--user`, to alter the user and group ID in the running container. It can be used in conjunction with the linux command `id` to pass the host IDs directly to the container. For instance:

```
> docker run -v `pwd`:/data -w /data -u `id -u`:`id -g` ubuntu touch container3
```

Now, let us inspect the ownerships of all the files created so far through containers:

```
> ls -l container?
-rw-r--r-- 1 root   root   0 Dec 19 08:16 container1
-rw-r--r-- 1 root   root   0 Dec 19 08:19 container2
-rw-r--r-- 1 ubuntu ubuntu 0 Dec 19 08:38 container3
```

As desired, the last created file is owned by the host user.

Finally, note that when running a container interactively with host IDs, you might get warnings of this type:

```
> docker run -it -u `id -u`:`id -g` ubuntu bash
groups: cannot find name for group ID 1000
I have no name!@9bfdf83aed93:/data$

> exit
```

These can typically be ignored.


### Conclusion ###
Docker provides flags to map host directories in the containers, and to match file ownerships. The syntax looks ugly, but once learnt it can be reused with minimal variations.


### Best practices ###

- figuring out a standard way to consistently map host directories in container can help scripting and automation. For instance:
    - ``` -v `pwd`:/data -w /data ``` can be useful when just working in the current directory
    - ``` -v /<DATA-DIRECTORY> -w /data ``` can be useful if your workstation/cluster is organised with one directory called `<DATA-DIRECTORY>` that contains all sample data and reference data
- eventually, multiple volume mappings are allowed at the same time, for instance:
        ```-v `pwd`:/data -v /reference-database:/ref -w /data ```