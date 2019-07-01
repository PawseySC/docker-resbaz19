---
title: "Docker advanced topics"
teaching: 15
exercises: 5
questions:
objectives:
- Learn how to set custom user and group IDs in a container
- Learn how to redirect input to a container
- Learn how to use Docker Compose to manage multiple containers for web services
keypoints:
- "Change user that runs the container with the flag `-u <user>:<group>`"
- "Make the container accept input from STDIN with the flag `-i`"
- "You can use a Docker Compose YAML file to orchestrate the setup of multiple containers at once"
---

### Input redirection with Docker ###

Suppose you need to redirect input to a Docker container, either using an input file with `<` or piping from another execution with `|`. If you just run the container as usual you won't get the expected result. As an example, let's use the unix command `wc -w` to count the number of words received in input; we'll produce the words with `echo`:

```
$ echo "one two three" | docker run ubuntu wc -w
```
{: .bash}

```
0
```
{: .output}

We are getting `0` instead of `3`, suggesting input redirection is not happening. To make it work, we need to use the additional flag `-i`, which keeps Docker `STDIN` (standard input) open:

```
$ echo "one two three" | docker run -i ubuntu wc -w
```
{: .bash}

```
3
``` 
{: .output}

Here we go!

The same flag is required when receiving input from a file, for instance let's create a file with some words:

```
$ echo "one two three" > words
```
{: .bash}

and then try and count them:

```
$ docker run ubuntu wc -w < words
```
{: .bash}

```
0
```
{: .output}

```
$ docker run -i ubuntu wc -w < words
```
{: .bash}

```
3
```
{: .output}


### Matching user permissions with the host ###

We have seen in a previous episode how files created by the container belong to the root user. This can be annoying, in that the host user might then have limitations in editing/deleting those files. 
Docker has an option, `-u` or `--user`, to alter the user and group ID in the running container. It can be used in conjunction with the linux command `id` to pass the host user and group IDs directly to the container. For instance:

```
$ docker run -v `pwd`:/data -w /data -u `id -u`:`id -g` ubuntu touch container3
```
{: .bash}

Now, let us inspect the ownerships of all the files created so far through containers:

```
$ ls -l container?
```
{: .bash}

```
-rw-r--r-- 1 root   root   0 Dec 19 08:16 container1
-rw-r--r-- 1 root   root   0 Dec 19 08:19 container2
-rw-r--r-- 1 ubuntu ubuntu 0 Dec 19 08:38 container3
```
{: .output}

As desired, the last created file is owned by the host user.

When running a container interactively with host IDs, you might get warnings of this type:

```
$ docker run -it -u `id -u`:`id -g` ubuntu bash
```
{: .bash}

```
groups: cannot find name for group ID 1000
I have no name!@9bfdf83aed93:/data$
```
{: .output}

These can typically be ignored.

```
I have no name!@9bfdf83aed93:/data$ exit   # or hit CTRL-D
```
{: .bash}

Finally, third-party containers might have been set-up so that permissions of standard users are more restricted compared to root. There are some critical cases here: only root can execute the application, or only root can write on certain locations that need to be modified at runtime. In these cases, the typical solutions are:
* run the container as root, then fix output file ownerships;
* build your own container for that application, with appropriate privileges for non-root users.


### Long running services with Docker Compose ###

Sometimes we may need to run and manage multiple containers (e.g., running an Nginx web-server in front of some other application we are running in a container).

We can run all of these via `docker run` commands, but it may get cumbersome if you have lots of arguments and flags.  We can use a tool called `docker-compose` to orchestrate and manage multiple containers for us.

All we need to do is define the various options and properties for our containers in a YAML file.  Let's create a simple, containerised MySQL/Nginx setup:

<!--
```
version: '3'
services:

  # Nginx
  webserver:
    image: nginx:alpine
    container_name: nginx-webserver
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    networks:
      - appnet

  # MySQL
  db:
    image: mysql:5.7.22
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: mydb
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - dbdata:/var/lib/mysql
    networks:
      - appnet

  # Docker Volumes
  volumes:
    dbdata:
      driver:local
     
  # Docker Networks
  networks:
    appnet:
      driver: bridge
```
{: .source}
-->

```
version: '3'
services:

  # Nginx
  webserver:
    image: nginx:alpine
    container_name: nginx-webserver
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"

  # MySQL
  db:
    image: mysql:5.7.22
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: mydb
      MYSQL_ROOT_PASSWORD: password
    volumes:
      - <PICK-A-HOST-DIRECTORY>:/var/lib/mysql
```
{: .source}

Here we can define different services and options.  There are a lot of options, so I'll touch on a few:

* image - this is the Docker image you want to pull
* restart - we can tell docker-compose to restart containers unders certain conditions
* ports - open up different ports to a container
* networks - we define a virtual network for containers to connect to
* volumes - Docker volumes are persistent data stores we can use to store app data after a container ends

To run this, you simply need to save the above file as `docker-compose.yml`, cd to that directory and run:

```
$ docker-compose up -d
```
{: .bash}

To shut it down, from the same directory run:

```
$ docker-compose down
```
{: .bash}

Another example of usage for `docker-compose` can be found at this Github repo: <https://github.com/PawseySC/rstudio-nginx>.

The YAML file in this repo sets up a secure RStudio server using Nginx.
