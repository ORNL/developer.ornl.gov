---
title: Using Containers across OLCF
author: David M. Rogers
tags: systems packaging
---

Containers have proven to be a popular technology for making software reproducible.  As the name suggests, they build a wall around your program package, providing some level of isolation from the system environment.[^1]  This separation is like a `chroot` - creating a new filesystem just for the container's working space.  All the container's processing is still making use of the same kernel on the host operating system.  However, all program file accesses, library loads, and executables are addressed within the container's working directory space.

A killer feature of containers is that they can be checkpointed to image files, which store a frozen, immutable, state of the applications that were running in that container.  As developers got used to this technology, they started to stack containers together, with one logical process running in each container, to make `compositions' of containers.

![Custom-designed crate moving a million dollar storage cabinet attached to OLCF Titan](/images/MovingEquipmentM.jpg)

## Mini List of Container Categories for HPC

* [E4S Base Images](https://e4s-project.github.io/download.html)
* [NVHPC HPL](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/nvhpc)
* [NVIDIA AI](https://developer.nvidia.com/ai-hpc-containers)
* [AMD CP2K](https://www.amd.com/en/technologies/infinity-hub/cp2k)


## Docker vs Apptainer(Singularity) vs CharlieCloud

The idea of containers that can be checkpointed to images can be implemented in many different ways.  Some of the more important differences are how external resources are accessed within the container.  In Docker's default configuration, it is possible to mount host filesystems and access priviledged network ports within the container.  This essentially gives you root access to the host system.  This are not desirable features on shared compute systems.

Apptainer and CharlieCloud are new implementations for container run-times (along with different build file and image file formats).
In Apptainer, isolation from the host is maintained through advanced interaction with the kernel and filesystem.  These include mapping your regular user-name on the host to root on the container [as a fake root](https://sylabs.io/guides/3.7/admin-guide/user_namespace.html).  Apptainer can also be installed by a regular user.  Docker has done some work to add similar features, but still leans more on the side of sandboxed root execution than user-level execution for containers.
CharlieCloud takes the opposite approach and attempts to minimally isolate your container from the host.  Your user on the host is the user on the container, and host resource mapping is straightforward.  Using minimal isolation like this allows you to run containers as you would normally (e.g. load modules on HPC systems), but still use images as checkpoints.


## Deploying an Image (creating a Container)

Once an image has been created by someone, somewhere, you can `run` it. 
The running image is called a container.

For example, we can deploy the ROCM base system image from [https://e4s-project.github.io/download.html]

    wget https://oaciss.uoregon.edu/e4s/images/22.02/e4s-x86_64-rocm.sif
    singularity run --rocm e4s-x86_64-rocm.sif # note: flag is --nv if you are using NVIDIA devices instead of AMD.

Note that apptainer is already installed in `/usr/bin/singularity` on OLCF systems.

## Packaging a Container

There is an important distinction between images and build-files for images.  Build-files for images are a container's '''source-code''', while images are their '''compiled binaries'''.  Different platforms have different file formats.  For docker, build files are usually called `Dockerfile`.  For an example `Dockerfile`, see [fastapi's documentation](https://fastapi.tiangolo.com/deployment/docker/).  It includes an example of both building a container from a "base image" containing Ubuntu with python and building a container from a "base image" containing Ubuntu with the fastapi library itself.  Docker unfortunately makes it easy to deploy uncheckable binaries this way because you start a dockerfile by loading an image.

I'll provide an example here of a build file (def file in Apptainer jargon[^2]) for Apptainer,

```
# mysql.def, an Apptainer Image Definition File for running MySQL
BootStrap: docker
From: amd64/alpine:3.16

%post
    apk add --no-cache mysql-client

%environment
    export LC_ALL=C

%runscript
    mysql

%labels
    Author David M. Rogers
```

Because `root = uid 0` is hard-coded into a lot of system utilities, creating a container often requires executing commands as root within the container.  Technically, this creates files inside your container space owned by the root user - which many systems do not permit.  In order to get around this restriction, I use the [syslabs builder](https://cloud.sylabs.io/builder).  Navigate to that website, create an account, and then run (at your command-line) `singularity remote login`.  This will direct you to visit a URL and generate a token.  Copy the token into the waiting terminal and hit enter.

Next, build with,

    $ singularity build --remote mysql.sif mysql.def 

Now you can run the `mysql` client from inside the container.  However, unless you have a running mysqld server, it's not likely
to do anything interesting.  Instead, let's run some diagnostics:

    $ singularity exec mysql.sif mysql --help
    /usr/bin/mysql  Ver 15.1 Distrib 10.6.8-MariaDB, for Linux (x86_64) using readline 5.1
    Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

    Usage: /usr/bin/mysql [OPTIONS] [database]

    $ singularity exec mount
    $ singularity exec mysql.sif mount
    overlay on / type overlay (ro,nodev,relatime,lowerdir=/var/singularity/mnt/session/overlay-lowerdir:/var/singularity/mnt/session/rootfs)
    devtmpfs on /dev type devtmpfs (rw,nosuid,noexec,size=263539356k,nr_inodes=65884839,mode=755)
    devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
    tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,noexec)
    mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
    hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,pagesize=2M)
    overlay on /etc/localtime type overlay (rw,nosuid,nodev,relatime,lowerdir=/root_ro,upperdir=/rootfs.rw/upperdir,workdir=/rootfs.rw/work)
    overlay on /etc/hosts type overlay (rw,nosuid,nodev,relatime,lowerdir=/root_ro,upperdir=/rootfs.rw/upperdir,workdir=/rootfs.rw/work)
    overlay on /host_lib64 type overlay (rw,nosuid,nodev,relatime,lowerdir=/root_ro,upperdir=/rootfs.rw/upperdir,workdir=/rootfs.rw/work)
    proc on /proc type proc (rw,nosuid,nodev,relatime)
    sysfs on /sys type sysfs (rw,nosuid,nodev,relatime)
    tmpfs on /tmp type tmpfs (rw,nosuid,nodev,relatime,mpol=interleave:0-3)
    overlay on /var/tmp type overlay (rw,nosuid,nodev,relatime,lowerdir=/root_ro,upperdir=/rootfs.rw/upperdir,workdir=/rootfs.rw/work)
    tmpfs on /etc/resolv.conf type tmpfs (rw,nosuid,relatime,size=16384k,uid=1001,gid=100)
    tmpfs on /etc/passwd type tmpfs (rw,nosuid,relatime,size=16384k,uid=1001,gid=100)
    tmpfs on /etc/group type tmpfs (rw,nosuid,relatime,size=16384k,uid=1001,gid=100)
    11.22.33.44:/nfs_proj on /proj type nfs (rw,nosuid,nodev,relatime)

Many of the regular system paths (notably /) are replaced with overlay and tmpfs.
Note how the NFS mount is still present in the container.
Apptainer has a special rule not to expose the home directory to the container,
but it does expose other mounted filesystems.


The simple example above shows a minimal build that you can add to.
For a more advanced recipe building a singularity container for tensorflow on NVIDIA GPUs,
see [this recipe](https://github.com/jdongca2003/Tensorflow-singularity-container-with-GPU-support).
It provides a great example for an image build process, since it shows both the image's definition
file _and_ shell scripts that build it, along with some helpful user documentation.


## Summary

Containers provide a means for bundling up dependency packages so that users can quickly reproduce your computations.
One of their main uses is to spare users (and yourself) from the hassle of installing all project build dependencies
and setting up environmental files in a specific way.

A side-benefit is the ability to checkpoint and restart a filesystem state.  This can be important for
things like runing your project's correctness-tests, or doing builds inside continuous integration.

Two main classes of applications we see a lot at OLCF are python utilities and GPU+MPI intensive C++ codes.
Build instruction files for python can be as simple as pip-installing several things to make
a starting image.  For MPI and GPU-intensive codes, Apptainer provides `--nv` and `--rocm` flags
that load the libraries and executables present in `/etc/singularity/*liblist.conf` into your container
from the host.  These allow the Apptainer tensorflow example to build and run.  For codes that use MPI,
it would be great work on a similar mechanism!

# References

[^1] [Early History of Docker](https://www.docker.com/blog/5-years-later-docker-come-long-way/)

[^2] [Apptainer Definition File](https://sylabs.io/guides/latest/user-guide/definition_files.html)
