---
title: Using Containers across OLCF
author: David M. Rogers
tags: systems packaging
---

Containers have proven to be a popular technology for making software reproducible.  As the name suggests, they build a wall around your program package, providing some level of isolation from the system environment.  This separation is like a `chroot` - creating a new filesystem just for the container's working space.  All the container's processing is still making use of the same kernel on the host operating system.  However, all program file accesses, library loads, and executables are addressed within the container's working directory space.

A killer feature of containers is that they can be checkpointed to image files, which store a frozen, immutable, state of the applications that were running in that container.  As developers got used to this technology, they started to stack containers together, with one logical process running in each container, to make `compositions' of containers.[^1]

![Custom-designed crate moving a million dollar storage cabinet attached to OLCF Titan](/images/MovingEquipmentM.jpg)

There are many good guides and resource for containers out there.[^2]
This article will focus on how we use containers here at OLCF.

## Mini List of Container Categories for HPC

First up, here are a few demonstrations of containerized killer applications:

* [E4S Base Images](https://e4s-project.github.io/download.html)
* [NVHPC HPL](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/nvhpc)
* [NVIDIA AI](https://developer.nvidia.com/ai-hpc-containers)
* [AMD CP2K](https://www.amd.com/en/technologies/infinity-hub/cp2k)

To use containers like these on our systems, we have installed singularity (in `/usr/bin/singularity`) on most of the supercomputers at the Oak Ridge Leadership Computing Facility (OLCF).

### Digression - Comparing Docker vs Apptainer(Singularity) vs CharlieCloud

The idea of containers and their image file formats can be implemented in many different ways.  Some of the more important differences are how external resources are accessed within the container.  In Docker's default configuration, it is possible to mount host filesystems and access priviledged network ports within the container.  This essentially gives you root access to the host system -- which is not desirable on shared compute systems.

Apptainer and CharlieCloud are new implementations for container run-times (along with different build file and image file formats).
In Apptainer, isolation from the host is maintained through advanced interaction with the kernel and filesystem.  These include mapping your regular user-name on the host to root on the container [as a fake root](https://sylabs.io/guides/3.7/admin-guide/user_namespace.html).  Apptainer can also be installed by a regular user.  Docker has done some work to add similar features, but still leans more on the side of sandboxed root execution than user-level execution for containers.
CharlieCloud takes the opposite approach and attempts to minimally isolate your container from the host.  Your user on the host is the user on the container, and host resource mapping is straightforward.  Using minimal isolation like this allows you to run containers as you would normally (e.g. load modules on HPC systems), but still use images as checkpoints.


## Deploying an Image (creating a Container)

Once an image has been created by someone, somewhere, you can `run` it
on a compatible host.  Compatibility is loosely determined by:

  1. whether the host's CPU architecture can run the executables in the image, and
  2. whether the device libraries used by those executables match the hardware available on the runtime-system
  
The running image is called a container.

For example, we can deploy the ROCM base system image from [E4S](https://e4s-project.github.io/download.html)

    wget https://oaciss.uoregon.edu/e4s/images/22.02/e4s-x86_64-rocm.sif
    module load rocm
    singularity run --rocm e4s-x86_64-rocm.sif # note: flag is --nv if you are using NVIDIA devices instead of AMD.


## Packaging a Container

There is an important distinction between images and build-files for images.  Build-files for images are a container's ***source-code***, while images are their ***compiled binaries***.  Different platforms have different file formats.  For docker, build files are usually called `Dockerfile`.  For an example `Dockerfile`, see [fastapi's documentation](https://fastapi.tiangolo.com/deployment/docker/).  It includes an example of both building a container from a "base image" containing Ubuntu with python and building a container from a "base image" containing Ubuntu with the fastapi library itself.  Docker unfortunately makes it easy to deploy uncheckable binaries this way because you start a dockerfile by loading an image.

I'll provide an example here of a build file (def file in Apptainer jargon[^3]) for Apptainer,

```
# pmake.dev, an Apptainer Image Definition File for installing pmake
BootStrap: docker
From: amd64/python:3.9

%post
    python -m pip install --no-cache-dir git+https://code.ornl.gov/99R/pmake.git@latest

%environment
    export LC_ALL=C

%runscript
    python3

%labels
    Author David M. Rogers
```

Because `root = uid 0` is hard-coded into a lot of system utilities, creating a container often requires executing commands as root within the container.  Technically, this creates files inside your container space owned by the root user - which many systems do not permit.  In order to get around this restriction, I use the [syslabs builder](https://cloud.sylabs.io/builder).  Navigate to that website, create an account, and then run (at your command-line) `singularity remote login`.  This will direct you to visit a URL and generate a token.  Copy the token into the waiting terminal and hit enter.

Next, build with,

    $ singularity build --remote pmake.sif pmake.def 

Now you can import pmake and/or run the `pmake` command from inside the container.
To see what's happened, let's run some diagnostics:

    $ singularity exec pmake.sif pmake --help
    
    Usage: pmake [OPTIONS] [RULES] [TARGETS] [MINUTES]

    Arguments:
      [RULES]    Make rules.  [default: rules.yaml]
      [TARGETS]  Make targets.  [default: targets.yaml]
      [MINUTES]  Number of minutes available to run steps.  [default: inf]

    Options:
      --workdir PATH                  Store all jobscripts and job logs in this
                                      directory (instead of each target's dir).
                                      [default: .]
      --log PATH                      Output a debug-level logfile.
      --test / --no-test              List jobs, but do not create jobfiles or run
                                      anything.  [default: no-test]
      --install-completion [bash|zsh|fish|powershell|pwsh]
                                      Install completion for the specified shell.
      --show-completion [bash|zsh|fish|powershell|pwsh]
                                      Show completion for the specified shell, to
                                      copy it or customize the installation.
      --help                          Show this message and exit.

    $ singularity exec pmake.sif mount
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
To jump directly to a super advanced recipe, building a singularity container for tensorflow on NVIDIA GPUs,
see [this recipe](https://github.com/jdongca2003/Tensorflow-singularity-container-with-GPU-support).
It provides a great example for an image build process, since it shows both the image's definition
file _and_ shell scripts that build it, along with some helpful user documentation.


## Containers with GPU and MPI on OLCF systems

Here's the tricky part - getting your containers to use the hardware-specific MPI and GPU libraries
on Summit, Crusher, and Frontier.
Luckily for us, OLCF has a group of system experts that have provided us "best practices" recipes
for making this happen.

On Summit, Subil Abraham documented the process in the
[Containers on Summit](https://docs.olcf.ornl.gov/software/containers_on_summit.html) guide.

He builds using `podman` on Summit, then converts the resulting images to sif format for use with Apptainer.
When running the containers, the following environment variables are used to load libraries from the
host OS into the container:

  * `SINGULARITY_CONTAINLIBS`: A comma-separated string of file paths bound to the /.singularity.d/libs directory inside the container

  * `SINGULARITYENV_LD_PRELOAD`: sets `LD_PRELOAD` inside the container (libraries from the host that should be pre-loaded into every running application)

  * `SINGULARITYENV_LD_LIBRARY_PATH`: sets `LD_LIBRARY_PATH` inside the container (library search path) 

His Docker recipes, output images, and run-scripts are all collected
on [code.ornl.gov/olcfcontainers/olcfbaseimages](https://code.ornl.gov/olcfcontainers/olcfbaseimages).


For Crusher and Frontier, Matthew Davis has provided the following recipe:
```
# Dockerfile
FROM almalinux:8.5-20220306 AS builder
 
RUN yum install -y https://repo.radeon.com/amdgpu-install/22.10.1/rhel/8.5/amdgpu-install-22.10.1.50101-1.el8.noarch.rpm && \
    yum install -y epel-release && \
    yum --enablerepo=powertools install -y rocm-dev5.1.1 && \
    yum install -y mpich mpich-devel gcc-c++
 
ENV PATH="/opt/rocm/bin:/usr/lib64/mpich/bin:${PATH}"
ENV ROCM_PATH /opt/rocm
ENV LD_LIBRARY_PATH "/opt/rocm/lib:/opt/rocm/lib64"
 
RUN curl -O https://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-5.9.tar.gz && \
    tar -zxf osu-micro-benchmarks-5.9.tar.gz && \
    pushd osu-micro-benchmarks-5.9 && \
    ./configure CC=mpicc \
            CXX=mpicc \
            CFLAGS="-g -I${ROCM_PATH}/include" \
            CXXFLAGS="-g -I${ROCM_PATH}/include" \
            LDFLAGS="-L${ROCM_PATH}/lib -lamdhip64" \
            --enable-rocm --with-rocm=${ROCM_PATH} \
            --prefix=/opt/omb && \
    make && \
    make install
 
RUN yum clean all && \
    rm -rf /osu-micro-benchmarks-5.9 /osu-micro-benchmarks-5.9.tar.gz
 
FROM almalinux:8.5-minimal-20220306
 
COPY --from=builder /opt/omb/libexec/osu-micro-benchmarks /app
```

He builds on a laptop (`docker build --progress=plain -t osu-rocm:nolibs -f crusher_base_el.docker .`),
exports with `docker save -o osu-rocm.tar osu-rocm`,
copies the result to crusher, then converts to sif with
`singularity build --disable-cache osu-rocm.sif docker-archive://osu-rocm.tar`.

To run, he creates a wrapper shell script to set up the environment and launch a
program inside the container, 
```
#!/bin/bash
# exec.sh - wrapper for running a singularity containerized application on crusher
 
ROCM_VER="5.0.2"
SIF="$1"
shift
 
module -q load PrgEnv-gnu
module -q load rocm/${ROCM_VER}
module -q load cray-mpich-abi/8.1.15
 
export MPICH_GPU_SUPPORT_ENABLED=1
 
export SINGULARITYENV_LD_LIBRARY_PATH="/opt/cray/pe/lib64:$LD_LIBRARY_PATH"
export SINGULARITYENV_LD_PRELOAD="/opt/cray/pe/mpich/8.1.15/gtl/lib/libmpi_gtl_hsa.so.0"
export SINGULARITY_CONTAINLIBS="/usr/lib64/libcxi.so.1,/usr/lib64/libjson-c.so.3,/usr/lib64/libdrm_amdgpu.so.1,/usr/lib64/libdrm.so.2,/lib64/libtinfo.so.6"
 
singularity exec \
    --rocm \
    --bind $ROCM_PATH \
    --bind /usr/share/libdrm \
    --bind /var/spool/slurm \
    --bind /opt/cray \
    --env MV2_COMM_WORLD_LOCAL_RANK="$SLURM_LOCALID" \
    --bind ${PWD} \
    --workdir ${PWD} \
    ${SIF} $@
```

This is launched inside a Slurm job using:

    srun -N2 -n8 ./exec.sh osu-rocm.sif /app/get_local_rank /app/mpi/collective/osu_allgather -d rocm

Further, he provides some general guidance:

  * If the container is providing its own ROCm, that version must be compatible with the chosen GTL host library (i.e. the LD_PRELOAD one)
  * If the container is not providing ROCm, the application built with ROCm should be built with a version of ROCm that is ABI compatible with the host ROCm chosen. (that's what the above accomplishes, so rocm-dev5.1.1 appears ABI compatible with our rocm-5.0.2 module)
  * AMD does not increment the SONAME to match ABI compatibility, but instead ties it to the release version. This makes compatibility issues more difficult to diagnose immediately.
  * If using the host MPI libraries, the container application should have been built with an ABI compatible MPI. (libmpi.so.12 here)

## Summary

Containers provide a means for bundling up dependency packages so that users can quickly reproduce your computations.
One of their main uses is to spare users (and yourself) from the hassle of installing all project build dependencies
and setting up environmental files in a specific way.

A side-benefit is the ability to checkpoint and restart a filesystem state.  This can be important for
things like running your project's correctness-tests, or doing builds inside continuous integration.

Two main classes of applications we see a lot at OLCF are GPU+MPI intensive C++ codes and
some python utilities.
The first example above showed that build instruction files for python can be as simple as pip-installing
several things to make a starting image.

For GPU-intensive codes, Apptainer provides `--nv` and `--rocm` flags
that load the libraries and executables present in `/etc/singularity/*liblist.conf` into your container
from the host.  They are not magic, however.  As we can see from the examples above,
they rely on building your containerized applications on top of ABI-compatible
versions of libraries on the host.  For Summit and Frontier, the base images above
accomplish this.


# Authors

[David M. Rogers](https://www.olcf.ornl.gov/directory/staff-member/david-rogers/) is a Computational Scientist in the Advanced Computing for Chemistry and Materials group, where he works to develop mathematical and computational theory jointly with methods for multiscale modeling using HPC. He obtained his Ph.D. in Physical Chemistry from University of Cincinnati in 2009 where he worked on applying Bayes' theorem to the free energy problem with applications to multiscale modeling of fluids and interface chemistry.

[Subil Abraham](https://www.olcf.ornl.gov/directory/staff-member/subil-abraham/) is an HPC Engineer in the Operations – User Assistance group. He previously worked as an intern in the Technology Integration group, testing SymphonyFS.  He received his MS in Computer Science from Virginia Tech in 2020. His thesis work involved evaluating container performance on HPC workloads and filesystems. His current interests include containers for HPC and parallel programming.

[Matthew Davis](https://www.olcf.ornl.gov/directory/staff-member/matthew-davis/) is an HPC Engineer in the System Acceptance and User Environment Group. Prior to joining ORNL he worked for the U.S. Geological Survey supporting a wide range of HPC activities, and for the Department of Defense administering and deploying hybrid cloud solutions. His academic studies were in pure mathematics, earning a B.S. in Mathematics from Purdue University and a M.A in Mathematics from the University of Missouri.

# References

[^1] [Early History of Docker](https://www.docker.com/blog/5-years-later-docker-come-long-way/)

[^2] BSSW.io, Exploring Containers for Research Software, Chris Richardson

[^3] [Apptainer Definition File](https://sylabs.io/guides/latest/user-guide/definition_files.html)
