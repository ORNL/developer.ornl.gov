---
title: Using Containers across OLCF
author: David M. Rogers
tags: systems packaging
---

Containers have proven to be a popular technology for making software reproducible.  As the name suggests, they build a wall around your program package, providing some level of isolation from the system environment.  This separation is like a `chroot` - creating a new filesystem just for the container's working space.  All the container's processing is still making use of the same kernel on the host operating system.  However, all program file accesses, library loads, and executables are addressed within the container's working directory space.



## List of Useful Containers for HPC

* E4S Program Package
* E4S Base Images
* NVHPC HPL
* NVIDIA AI
* AMD CP2K

## Docker vs Singularity

## Deploying an Image (creating a Container)

Once an image has been created by someone, somewhere, you can `run` it. 
The running image is called a container.

For example, we can deploy the ROCM base system image from [https://e4s-project.github.io/download.html]

    wget https://oaciss.uoregon.edu/e4s/images/22.02/e4s-x86_64-rocm.sif

## Packaging a Container

Creating a container often requires executing commands as root within the container.  Technically, this creates files inside your container space owned by the root user - which many systems do not permit.  In order to get around this restriction, I use the [syslabs builder](https://cloud.sylabs.io/builder)


## Incorporating Containers into your Pre-Push / CI Testing

## Summary

Containers provide a means for bundling up dependency packages so that users can quickly reproduce your computations.
