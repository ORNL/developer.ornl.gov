---
title: Using Containers across OLCF
author: David M. Rogers
tags: systems packaging
---

Containers have proven to be a popular technology for making software reproducible.  As the name suggests, they build a wall around your program package, providing some level of isolation from the system environment.  This separation is like a `chroot` - creating a new filesystem just for the container's working space.  All the container's processing is still making use of the same kernel on the host operating system.  However, all program file accesses, library loads, and executables are addressed within the container's working directory space.

A killer feature of containers is that they can be checkpointed to image files, which store a frozen, immutable, state of the applications that were running in that container.  As developers got used to this technology, they started to stack containers together, with one logical process running in each container, to make `compositions' of containers.


## Mini List of Container Categories for HPC

* [E4S Base Images](https://e4s-project.github.io/download.html)
* [NVHPC HPL](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/nvhpc)
* [NVIDIA AI](https://developer.nvidia.com/ai-hpc-containers)
* [AMD CP2K](https://www.amd.com/en/technologies/infinity-hub/cp2k)


## Docker vs Singularity vs CharlieCloud

The idea of containers that can be checkpointed to images can be implemented in many different ways.  Some of the more important differences are how external resources are accessed within the container.  In Docker's default configuration, it is possible to mount host filesystems and access priviledged network ports within the container.  This essentially gives you root access to the host system.  This are not desirable features on shared compute systems.

Singularity and CharlieCloud are new implementations for container run-times (along with different build file and image file formats).
In Singularity, isolation from the host is maintained through advanced interaction with the kernel and filesystem.  These include mapping your regular user-name on the host to root on the container [as a fake root](https://sylabs.io/guides/3.7/admin-guide/user_namespace.html).  Singularity can also be installed by a regular user.  Docker has done some work to add similar features, but still leans more on the side of sandboxed root execution than user-level execution for containers.
CharlieCloud takes the opposite approach and attempts to minimally isolate your container from the host.  Your user on the host is the user on the container, and host resource mapping is straightforward.  Using minimal isolation like this allows you to run containers as you would normally (e.g. load modules on HPC systems), but still use images as checkpoints.


## Deploying an Image (creating a Container)

Once an image has been created by someone, somewhere, you can `run` it. 
The running image is called a container.

For example, we can deploy the ROCM base system image from [https://e4s-project.github.io/download.html]

    wget https://oaciss.uoregon.edu/e4s/images/22.02/e4s-x86_64-rocm.sif
    singularity run --rocm e4s-x86_64-rocm.sif # note: flag is --nv if you are using NVIDIA devices instead of AMD.

Note that singularity is already installed in `/usr/bin/singularity` on OLCF systems.

## Packaging a Container

There is an important distinction between images and build-files for images.  Build-files for images are a container's '''source-code''', while images are their '''compiled binaries'''.  Different platforms have different file formats.  For docker, build files are usually called `Dockerfile`.  For an example `Dockerfile`, see [fastapi's documentation](https://fastapi.tiangolo.com/deployment/docker/).  It includes an example of both building a container from a "base image" containing Ubuntu with python and building a container from a "base image" containing Ubuntu with the fastapi library itself.  Docker unfortunately makes it easy to deploy uncheckable binaries this way because you start a dockerfile by loading an image.

I'll provide an example here of a build file (def file in Singularity jargon) for Singularity,

```
# mysql.def, a Singularity Image Definition File for running MySQL
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
Note how the NFS mount is still present in the singularity container.
Singularity has a special rule not to expose the home directory to the container,
but it does expose other mounted filesystems.


## Incorporating Containers into your Pre-Push / CI Testing

Now for an important use case when developing scientific software - running tests when committing new
code.  When I get started coding a project, I usually install all its dependencies in a folder associated
with the project workspace.  However, those folders don't translate easily across systems, and it can be
difficult to remember the process you used for setting them up in the first place.
If, however, I were to install the dependencies into a container, then I could run the container
in 3 places: locally on my laptop, and on the HPC compute system, and on a CI pipeline provider.

Here's an example of how to do that for a toolkit I'm developing that uses MPI, OpenMP, Thrust,
and LLNL Conduit.

First, we explore the E4S base image for rocm:

    curl -O https://oaciss.uoregon.edu/e4s/images/22.05/e4s-base-rocm-22.05.sif
    # check the links above for the latest images
    $ singularity exec e4s-base-rocm-22.05.sif bash

    Inactive Modules:
      1) PrgEnv-cray     4) cray-libsci     7) craype                10) libfabric
      2) cce             5) cray-mpich      8) craype-network-ofi    11) perftools-base
      3) cray-dsmml      6) cray-pmi        9) craype-x86-trento     12) xpmem


    Lmod is automatically replacing "cray-mpich/8.1.16" with "mpich/4.0.2".

    Singularity> 

So, singularity has given be a shell prompt but inactivated all the HPC modules because of the path replacement that happens inside the container.
The original HPC system modules are visible, but not usable.
Let's try again with module purge.

    $ module purge
    $ singularity exec e4s-base-rocm-22.05.sif bash
    Singularity> module list

    Currently Loaded Modules:
      1) mpich/4.0.2

 

    Singularity> module avail

    ----------- /spack/share/spack/lmod/linux-ubuntu20.04-x86_64/Core ------------
       cmake/3.23.1    mpich/4.0.2 (L)

    -------------------------- /sw/crusher/modulefiles ---------------------------
       DefApps/default          rocm/4.5.0          rocm/5.1.0 (D)
       rocm/4.5.2               rocm/5.0.0          rocm/5.0.2
       ...
    
    Singularity> which hipcc
    /usr/bin/hipcc
    Singularity> which spack
    /spack/bin/spack

OK, now we have the intended base environment.  MPI is going to be difficult to use within the container,
and some rocm modules bleed through because `/sw` is still mounted.  However, the container
provides cmake, rocm, and spack.

After trying some setup commands locally, I can create the following image to get most of what I need
(all but MPI).

```
export SPACK_DISABLE_LOCAL_CONFIG=1
export SPACK_USER_CACHE_PATH=$PWD/bootstrap
#spack bootstrap root $PWD/bootstrap
#==> Error: Error writing to config file: '[Errno 30] Read-only file system: '/spack/etc/spack/cray''
spack spec -lINt conduit+python+hdf5+mpi
spack external find
conduit+python+hdf5+mpi
Singularity> ls /opt/rocm-5.1.1/rocthrust/
include  lib

```


## Summary

Containers provide a means for bundling up dependency packages so that users can quickly reproduce your computations.

# References

[1] [Early History of Docker](https://www.docker.com/blog/5-years-later-docker-come-long-way/)
[2] [Singularity Definition File](https://sylabs.io/guides/latest/user-guide/definition_files.html)
