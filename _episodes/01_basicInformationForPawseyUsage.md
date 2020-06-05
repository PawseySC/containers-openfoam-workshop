---
title: "Basic information for OpenFOAM containers at Pawsey"
teaching: 10
exercises: 05
questions:
- Basic information for OpenFOAM containers at Pawsey
objectives:
- Explain about the OpenFOAM containers maintained by Pawsey
- Explain containers MPI requirements to run at Pawsey
keypoints:
- A singularity image is a file that can be stored anywhere, but we reccommend to use some defined "policy" within your group
- OpenFOAM images maintained by pawsey are stored at `/group/singularity/pawseyRepository/OpenFOAM/`
- Containers with MPI applications need to be equipped with MPICH for running on Crays
---

<p>&nbsp;</p>

## Meaning of icons and colours (just for the context of this tutorial)

> ## Series of steps (commands) to be typed in users terminal (guided by instructor)
>
{: .discussion}

> ## Series of steps (commands) to be typed in users terminal by themselves (in breakout room)
>
{: .challenge}

> ## Collapsed blocks, with additional information, important parts of the scripts or pre-executed steps
> No action required
{: .solution}

> ## The rest are just blocks ...
>
{: .callout}

> ## ...                     with general information
>
{: .prereq}

## A. OpenFOAM containers maintained by Pawsey

> ## OpenFOAM Singularity images maintained by Pawsey are in the following directory:
> ~~~
> /group/singularity/pawseyRepository/OpenFOAM/
> ~~~
{: .callout}

> ## A.I List the content of our repository
> 1. To check which versions we currently maintain:
> 
>     ~~~
>     zeus-1:~> ls -lat /group/singularity/pawseyRepository/OpenFOAM
>     ~~~
>     {: .bash}
>     
>     ~~~
>     total 5855948
>     drwxrwsr-x  2 maali    pawsey0001       4096 May 13 13:59 .
>     -rwxr-x---+ 1 espinosa pawsey0001 1005383680 May  7 10:24 openfoam-2.4.x-pawsey.sif
>     -rwxr-xr-x+ 1 espinosa pawsey0001 1330900992 May  6 12:38 openfoam-v1912-pawsey.sif
>     -rwxr-xr-x+ 1 espinosa pawsey0001 1199386624 May  5 17:14 openfoam-7-pawsey.sif
>     -rwxr-xr-x  1 espinosa pawsey0001 1308553216 Apr  6 12:23 openfoam-v1812-pawsey.sif
>     drwxrwsr-x  3 maali    pawsey0001       4096 Mar  3 09:52 ..
>     -rwxr-xr-x+ 1 espinosa pawsey0001 1152237568 Feb 11 13:27 openfoam-5.x-pawsey.sif
>     ~~~
>     {: .output}
>
{: .discussion}

> ## A.II Perform a basic mini-test
> 
> 1. Load the Singularity module:
>
>    ~~~
>    zeus-1:~> module load singularity
>    ~~~
>    {: .bash}
> 
> 2. Choose one image (we like to save its name in a variable called `theImage`):
>
>    ~~~
>    zeus-1:~> theImage="/group/singularity/pawseyRepository/OpenFOAM/openfoam-v1912-pawsey.sif"
>    ~~~
>    {: .bash}
>
>  
> 3. Use singularity to execute the `icoFoam` solver inside the chose container and call its help message:
> 
>    ~~~
>    zeus-1:~> singularity exec $theImage icoFoam -help
>    ~~~
>    {: .bash}
>
>    ~~~
>    Usage: icoFoam [OPTIONS]
>    Options:
>      -case <dir>       Specify case directory to use (instead of the cwd)
>      -decomposeParDict <file>
>                        Use specified file for decomposePar dictionary
>      -dry-run          Check case set-up only using a single time step
>      -dry-run-write    Check case set-up and write only using a single time step
>      -parallel         Run in parallel
>      -postProcess      Execute functionObjects only
>      -doc              Display documentation in browser
>      -help             Display short help and exit
>      -help-full        Display full help and exit
>    
>    Transient solver for incompressible, laminar flow of Newtonian fluids.
>    
>    Using: OpenFOAM-v1912 (1912) - visit www.openfoam.com
>    Build: _f3950763fe-20191219
>    Arch:  LSB;label=32;scalar=64
>    ~~~
>    {: .output}
{: .discussion}

> ## Yes, you can also use your own containers
> For example, I keep my personal containers in:
>
> ~~~
> zeus-1:~> ls /group/pawsey0001/espinosa/singularity/myRepository/OpenFOAM/
> openfoam-7-mickey.sif      openfoam-v1912-esi.sif
> openfoam-7-foundation.sif
> ~~~
> {: .bash}
>
> And our group containers in:
>
> ~~~
> zeus-1:~> ls /group/pawsey0001/singularity/groupRepository/OpenFOAM/
> openfoam-5.x_CFDEM-pawsey.sif
> ~~~
> {: .bash}
{: .callout}

<p>&nbsp;</p>

## B. OpenFOAM containers need MPICH to run on Crays 

> ## Singularity Hybrid Mode for MPI applications
>
> - "Hybrid mode" execution is a way in which Singularity allows a containerised MPI application to **use the host MPI libraries** for better performance.
> - The only restriction for the hybrid mode to work is that host-MPI installation and container-MPI installation to be **ABI compatible**
> - (ABI=Application Binary Interface)
> - More information about the "hybrid mode" can be found in the [Singularity documentation](https://sylabs.io/guides/3.5/user-guide/mpi.html?highlight=hybrid%20mode)
>
{: .prereq}

<p>&nbsp;</p>

> ## Why do we prefer Hybrid-mode?
> - Because it gives better performance:
> - Running containerised MPI applications with the internal MPI is not the best approach.
> - For example, we have tested the solution of the channel395 tutorial (10 executions each) on a desktop computer  with 4 cores:
> - (All cases use the fileHandler collated; purgeWrite 10; writeFormat binary; runtimeModifiable false;) 
> 
> | Tutorial | Container | Mode | Avg.ClockTime |
> |----------|-----------|------|-----------|
> | channel395 | openfoam/openfoam7-paraview56:latest | Docker-internalOpenMPI | 1064.8 s|
> | channel395 | openfoam-7-foundation.sif | Singularity-internalOpenMPI | 806.4 s|
> | channel395 | openfoam-7-foundation.sif | Singularity-hybrid-HostOpenMPI | 787.2 s|
> | channel395 | pawsey/openfoam:7 | Docker-internalMPICH | 975.4 s|
> | channel395 | openfoam-7-pawsey.sif | Singularity-internalMPICH | 783.6 s|
> | channel395 | openfoam-7-pawsey.sif | Singularity-hybrid-HostMPICH | 779.2 s|
>
> - It is also the only way to run multi-node applications
>
{: .callout}

<p>&nbsp;</p>

> ## Why MPICH?
>
> - To achieve the best performance in a supercomputer, we want to run in hybrid mode with the optimised host MPI
> - In the case of Magnus, **Cray-MPI (ABI compatible to MPICH) is the only supported flavour**
> - Therefore, we decided to support containerised MPI applications compiled with MPICH
> - MPICH containers also run properly on Zeus
>
{: .prereq}

<p>&nbsp;</p>

> ## Main drawback: 
>
> - Developers' OpenFOAM containers equipped with OpenMPI would not run properly on Magnus (even after conversion to Singularity)
> - OpenFOAM containers need to be built from scratch with MPICH
>
{: .callout}

<p>&nbsp;</p>

> ## B.I Briefly check the settings at Pawsey
> 
> 1. You can check the definition of key variables for hybrid-mode execution (`SINGULARITY_BINDPATH` and `SINGULARITYENV_LD_LIBRARY_PATH`) with:
> 
>    ~~~
>    zeus-1:~> module show singularity
>    ~~~
>    {: .bash}
>
>    ~~~
>    ---------------------------------------------------------------------------------------------------------------
>       /pawsey/sles12sp3/modulefiles/devel/singularity/3.5.2.lua:
>    ---------------------------------------------------------------------------------------------------------------
>    help([[Sets up the paths you need to use singularity version 3.5.2]])
>    whatis("Singularity enables users to have full control of their environment. Singularity 
>    containers can be used to package entire scientific workflows, software and 
>    libraries, and even data.
>    
>    For further information see https://sylabs.io/singularity")
>    whatis("Compiled with gcc/4.8.5")
>    setenv("MAALI_SINGULARITY_HOME","/pawsey/sles12sp3/devel/gcc/4.8.5/singularity/3.5.2")
>    prepend_path("MANPATH","/pawsey/sles12sp3/devel/gcc/4.8.5/singularity/3.5.2/share/man")
>    prepend_path("PATH","/pawsey/sles12sp3/devel/gcc/4.8.5/singularity/3.5.2/bin")
>    setenv("SINGULARITYENV_LD_LIBRARY_PATH","/usr/lib64:/pawsey/intel/17.0.5/compilers_and_libraries/linux/mpi/intel64/lib")
>    setenv("SINGULARITY_BINDPATH","/astro,/group,/scratch,/pawsey,/etc/dat.conf,/etc/libibverbs.d,/usr/lib64/libdaplofa.so.2,/usr/lib64/libdaplofa.so.2.0.0,/usr/lib64/libdat2.so.2,/usr/lib64/libdat2.so.2.0.0,/usr/lib64/libibverbs,/usr/lib64/libibverbs.so,/usr/lib64/libibverbs.so.1,/usr/lib64/libibverbs.so.1.1.14,/usr/lib64/libmlx5.so,/usr/lib64/libmlx5.so.1,/usr/lib64/libmlx5.so.1.1.14,/usr/lib64/libnl-3.so.200,/usr/lib64/libnl-3.so.200.18.0,/usr/lib64/libnl-cli-3.so.200,/usr/lib64/libnl-cli-3.so.200.18.0,/usr/lib64/libnl-genl-3.so.200,/usr/lib64/libnl-genl-3.so.200.18.0,/usr/lib64/libnl-idiag-3.so.200,/usr/lib64/libnl-idiag-3.so.200.18.0,/usr/lib64/libnl-nf-3.so.200,/usr/lib64/libnl-nf-3.so.200.18.0,/usr/lib64/libnl-route-3.so.200,/usr/lib64/libnl-route-3.so.200.18.0,/usr/lib64/librdmacm.so,/usr/lib64/librdmacm.so.1,/usr/lib64/librdmacm.so.1.0.14")
se   tenv("SINGULARITY_CACHEDIR","/group/pawsey0001/espinosa/.singularity")
>    ~~~
>    {: .output}
>
> 2. And you can check the MPI version installed inside the container with:
> 
>    ~~~
>    zeus-1:~> theImage="/group/singularity/pawseyRepository/OpenFOAM/openfoam-v1912-pawsey.sif"
>    zeus-1:~> singularity exec $theImage mpiexec --version
>    ~~~
>    {: .bash}
>
>    ~~~
>    HYDRA build details:
>        Version:                                 3.1.4
>        Release Date:                            Fri Feb 20 15:02:56 CST 2015
>        CC:                              gcc    
>        CXX:                             g++    
>        F77:                             gfortran   
>        F90:                             gfortran   
>        Configure options:                       '--disable-option-checking' '--prefix=/usr' '--enable-fast=all,O3' '--cache-file=/dev/null' '--srcdir=.' 'CC=gcc' 'CFLAGS= -DNDEBUG -DNVALGRIND -O3' 'LDFLAGS= ' 'LIBS=-lpthread ' 'CPPFLAGS= -I/tmp/mpich-build/mpich-3.1.4/src/mpl/include -I/tmp/mpich-build/mpich-3.1.4/src/mpl/include -I/tmp/mpich-build/mpich-3.1.4/src/openpa/src -I/tmp/mpich-build/mpich-3.1.4/src/openpa/src -D_REENTRANT -I/tmp/mpich-build/mpich-3.1.4/src/mpi/romio/include'
>        Process Manager:                         pmi
>        Launchers available:                     ssh rsh fork slurm ll lsf sge manual persist
>        Topology libraries available:            hwloc
>        Resource management kernels available:   user slurm ll lsf sge pbs cobalt
>        Checkpointing libraries available:       
>        Demux engines available:                 poll select
>    ~~~
>    {: .output}
{: .discussion}

<p>&nbsp;</p>
