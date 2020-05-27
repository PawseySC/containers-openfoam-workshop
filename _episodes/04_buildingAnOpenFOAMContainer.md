---
title: "Building an OpenFOAM container with MPICH"
teaching: 30
exercises: 30
questions:
- How can I build my own OpenFOAM container with MPICH?
objectives:
- Explain our current recommended process for building OpenFOAM containers
keypoints:
- Take it easy and use existing `Dockerfile` examples and available guides to define your own recipe
- We recommend to used Docker for the recipe and the port to Singularity
---

<p>&nbsp;</p>

> ## Singularity Hybrid Mode for MPI applications
>
> - "Hybrid mode" execution is a way in which Singularity allows a containerised MPI application to **use the host MPI libraries** for better performance.
> - The only restriction for the hybrid mode to work is that host-MPI installation and container-MPI installation to be **ABI compatible**
> - (ABI=Application Binary Interface)
>
{: .prereq}

<p>&nbsp;</p>

> ## Why do we prefer Hybrid?
> - Because it gives better performance:
> - Running containerised MPI applications with the internal MPI is not the best approach.
> - For example, we have tested the solution of the channel395 tutorial (10 executions each) on a desktop computer  with 4 cores:
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
> - Developers' OpenFOAM containers equipped with OpenMPI would not run properly on Magnus
> - OpenFOAM containers need to be built from scratch with MPICH
>
{: .callout}

<p>&nbsp;</p>

## A. Have ready a Linux environment 
- If you count with a linux environment with Docker, Singularity and Git installed in your local computer, then you are ready to go to section B.

- If not, you can use the Nimbus Virtual Machines provided for the training:

> ## Steps for connecting to a Nimbus Virtual Machine:
>
> If you do not count with such a system, you can use one of the virtual machines provided for the training.
>
> Follow these steps to connect to your Nimbus virtual machine:
>
> 1. Save the ssh-key provided by the instructors in a known directory, for example `$HOME/pawseyTraining`:
>    ~~~
>    yourLocalComputer$ cd $HOME/pawseyTraining
>    yourLocalComputer$ ls
>    ~~~
>    {: .bash}
>    
>    ~~~
>    cfd_key.gz
>    ~~~
>    {: .output}
>    
>    ~~~
>    yourLocalComputer$ gunzip cfd_cfd2020.gz
>    ~~~
>    {: .bash}
>    
>    ~~~
>    yourLocalComputer$ ls -lat
>    ~~~
>    {: .bash}
>    
>    ~~~
>    drwxr-xr-x   4 esp025  515598196   128 26 May 08:11 .
>    drwxr-xr-x+ 75 esp025  515598196  2400 26 May 08:08 ..
>    -rw-r--r--   1 esp025  515598196  3434 22 May 11:26 rsa_cfd2020
>    ~~~
>    {: .output}
>    
>    - The point of the process is for you to count with the file: `rsa_cfd2020` in a known location
>
> 2. Make sure that the ssh-key has read-write permissions **only for you**:
>
>    ~~~
>    yourLocalComputer$ chmod 600 rsa_cfd2020
>    yourLocalComputer$ ls -lat
>    ~~~
>    {: .bash}
>
>    ~~~
>    total 24
>    drwxr-xr-x   4 esp025  515598196   128 26 May 08:11 .
>    drwxr-xr-x+ 75 esp025  515598196  2400 26 May 08:08 ..
>    -rw-------   1 esp025  515598196  3434 22 May 11:26 rsa_cfd2020
>    ~~~
>    {: .output}
>
> 3. Connect to the VM using that ssh-key:
>
>    - Choose an IP from the list of available VMs given by the instructors.
>
>    - The chosen IP address should be something like `123.123.123.123`
>
>    - Now, use the ssh-key to connect to the VM with that IP address (username is `ubuntu`):
>
>    ~~~
>    yourLocalComputer$ ssh -i rsa_cfd2020 ubuntu@123.123.123.123 
>    ~~~
>    {: .bash}
>
>    ~~~
>    Enter passphrase for key 'rsa_cfd2020':
>    ~~~
>    {: .output}
>
>    ~~~
>    Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 5.3.0-53-generic x86_64)
>    
>     * Documentation:  https://help.ubuntu.com
>     * Management:     https://landscape.canonical.com
>     * Support:        https://ubuntu.com/advantage
>    
>      System information as of Tue May 26 00:31:58 UTC 2020
>    
>      System load:  0.0                Processes:              102
>      Usage of /:   33.9% of 77.36GB   Users logged in:        0
>      Memory usage: 2%                 IP address for ens3:    192.168.1.17
>      Swap usage:   0%                 IP address for docker0: 172.17.0.1
>    
>     * MicroK8s passes 9 million downloads. Thank you to all our contributors!
>    
>         https://microk8s.io/
>    
>    0 packages can be updated.
>    0 updates are security updates.
>    
>    Your Hardware Enablement Stack (HWE) is supported until April 2023.
>    
>    Last login: Mon May 25 23:18:39 2020 from 27.33.35.111
>    ubuntu@vm:~$ 
>    ~~~
>    {: .output}
>
{: .discussion}

<p>&nbsp;</p>

## B. Clone the Git repository for the workshop exercises

You'll need to clone the git repository into your linux environment:

> ## Steps for cloning the Git repository
>
> 1. Create a directory for the training and clone the repository
> 
>    ~~~
>    ubuntu@vm:~$ mkdir pawseyTraining
>    ubuntu@vm:~$ cd pawseyTraining
>    ubuntu@vm:pawseyTraining$ git clone https://github.com/PawseySC/containers-openfoam-workshop-scripts.git
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Cloning into 'containers-openfoam-workshop-scripts'...
>    remote: Enumerating objects: 30, done.
>    remote: Counting objects: 100% (30/30), done.
>    remote: Compressing objects: 100% (16/16), done.
>    remote: Total 30 (delta 10), reused 30 (delta 10), pack-reused 0
>    Unpacking objects: 100% (30/30), done.
>    ~~~
>    {: .output}
>
>    ~~~
>    ubuntu@vm:pawseyTraining$ ls
>    ~~~
>    {: .bash}
>
>    ~~~
>    containers-openfoam-workshop-scripts
>    ~~~
>    {: .output}
>
>
{: .challenge}

<p>&nbsp;</p>

## C. Quick look into the Pawsey's repositories

- Pawsey maintains several images that can be used for running applications or for building new ones

- The GitHub repository (where the definition files are shared) is: [https://github.com/PawseySC/pawsey-containers](https://github.com/PawseySC/pawsey-containers)

- The DockerHub repository (where the corresponding Docker images are) is: [https://hub.docker.com/u/pawsey](https://hub.docker.com/u/pawsey)

<p>&nbsp;</p>

## D. Building a first Docker image with MPICH

- As mentioned in the previous days, builiding containers of large applications will need some trial-and-error process

- The building of an OpenFOAM image needs trial-and-error. (Installations of OpenFOAM are sometimes very time consuming)

- Therefore, we recommended to build the image first with Docker and convert it to Singularity afterwards

- This also allows more portability

> ## Steps for building our first image based on `pawsey/mpich-base`
> 1. cd into the exercise directory:
>    ~~~
>    ubuntu@vm:pawseyTraining$ cd containers-openfoam-workshop-scripts/04_buildingAnOpenFOAMContainer/openfoam-2.4.x 
>    ubuntu@vm:openfoam-2.4.x$ ls 
>    ~~~
>    {: .bash}
>
>    ~~~
>    01_Docker  02_PortingToSingularity
>    ~~~
>    {: .output}
>    - The 01_Docker directory contains the definition files for building the Docker image
>    - The 02_PortingToSingularity directory contains the definition file for building the Singularity image
>
>    <p>&nbsp;</p>
>
>    ~~~
>    ubuntu@vm:openfoam-2.4.x$ cd 01_Docker
>    ubuntu@vm:01_Docker$ ls
>    ~~~
>    {: .bash}
>
>    ~~~
>    Dockerfile.01  Dockerfile.02
>    ~~~
>    {: .output}
>
> 2. Let's take a look into the first Dockerfile (`Dockerfile.01`):
>
>    > ## Main sections of the `Dockerfile.01` to discuss:
>    >
>    > ~~~
>    > # 0. Initial main definintion
>    > # Defining the base container to build from
>    > # In this case: mpich 3.1.4 and ubuntu 16.04; MPICH is needed for crays
>    > FROM pawsey/mpich-base:3.1.4_ubuntu16.04
>    > 
>    > LABEL maintainer="Alexis.Espinosa@pawsey.org.au"
>    > #OpenFOAM version to install
>    > ARG OFVERSION="2.4.x"
>    > #Using bash from now on
>    > SHELL ["/bin/bash", "-c"]
>    > ~~~
>    > {: .bash}
>    > - Will build from the mpich-base image with MPICH-3.1.4 on Ubuntu-16.04
>    > - The "docker-variable" (argument) `OFVERSION` holds the version of OpenFOAM to be built
>    >
>    > <p>&nbsp;</p>
>    >
>    > ~~~
>    > # I. Installing additional tools useful for interactive sessions
>    > RUN apt-get update -qq\
>    >  &&  apt-get -y --no-install-recommends install \
>    >             vim time\
>    >             cron gosu \
>    >             bc \
>    >  && apt-get clean all \
>    >  && rm -r /var/lib/apt/lists/*
>    > ~~~
>    > {: .bash}
>    > - Using apt-get to install some auxiliary tools
>    >
>    > <p>&nbsp;</p>
>    >
>    > ~~~
>    > # III. INSTALLING OPENFOAM.
>    > #This section is for installing OpenFOAM
>    > #Will follow PARTIALLY the official installation instructions:
>    > #https://openfoam.org/download/2-4-0-source/
>    > #
>    > #Will follow PARTIALLY the instructions for OpenFOAM-2.4.x  available in the wiki:
>    > #https://openfoamwiki.net/index.php/Installation/Linux/OpenFOAM-2.4.x/Ubuntu#Ubuntu_16.04
>    > #
>    > #Then, Will follow a combination of both
>    > #(Also checking wiki for OpenFOAM-2.4.0)
>    > #
>    > #Where recipe deviates from the instructions mentioned above, comments from the maintainer are labelled as: AEG
>    > 
>    > #...........
>    > #Definition of the installation directory within the container
>    > ARG OFINSTDIR=/opt/OpenFOAM
>    > ARG OFUSERDIR=/home/ofuser/OpenFOAM
>    > WORKDIR $OFINSTDIR
>    > ~~~
>    > {: .bash}
>    > - Beginning of the main section for the OpenFOAM installation
>    > - Here we keep some comments about the guides used for defining this recipe
>    > - In this case: [https://openfoam.org/download/2-4-0-source/](https://openfoam.org/download/2-4-0-source/)
>    > - and [https://openfoamwiki.net/index.php/Installation/Linux/OpenFOAM-2.4.x/Ubuntu#Ubuntu_16.04](https://openfoamwiki.net/index.php/Installation/Linux/OpenFOAM-2.4.x/Ubuntu#Ubuntu_16.04)
>    > - Differences from the guides are noted by the maintainer with comments starting with `AEG`
>    > - We are also defining the path where the installation will reside (`OFINSTDIR`)
>    > - And the path where theoretical user stuff will reside (`OFUSERDIR`)
>    > - **`WORKDIR`** indicates the builder to create that directory and to move into it
>    >
>    > <p>&nbsp;</p>
>    >
>    > ~~~
>    > #...........
>    > #Step 1.
>    > #Install necessary packages
>    > #
>    > RUN apt-get update -qq\
>    >  &&  apt-get -y --no-install-recommends --no-install-suggests install \
>    >    build-essential\
>    >    flex bison git-core cmake zlib1g-dev \
>    >    libboost-system-dev libboost-thread-dev \
>    > #AEG:No OpenMPI because MPICH will be used (installed in the parent FROM container)
>    > #AEG:NoOpenMPI:   libopenmpi-dev openmpi-bin \
>    > #AEG:(using libncurses-dev, as in official instructions, and not libncurses5-dev, as in wiki)
>    >    gnuplot libreadline-dev libncurses-dev libxt-dev \
>    >    qt4-dev-tools libqt4-dev libqt4-opengl-dev \ 
>    >    freeglut3-dev libqtwebkit-dev \
>    > #AEG:No scotch because it installs openmpi which later messes up with MPICH
>    > #    Therefore, ThirdParty scotch is the one to be installed and used by openfoam.
>    > #AEG:NoScotch:   libscotch-dev \
>    >    libcgal-dev \
>    > #AEG:These libraries are needed for CGAL (system and third party) (if needed, change libgmp-dev for libgmp3-dev):
>    >    libgmp-dev libmpfr-dev\
>    > #AEG: Some more suggestions from the wiki instructions:
>    >    python python-dev \
>    >    libglu1-mesa-dev \
>    > #AEG:I found the following was needed to install  FlexLexer.h
>    >    libfl-dev \
>    >  && apt-get clean all \
>    >  && rm -r /var/lib/apt/lists/*
>    > ~~~
>    > {: .bash}
>    > - Here we install the auxiliary packages for OpenFOAM listed in the instructions guides
>    > - With some differences (check comments by `AEG`)
>    > - Like: **OpenMPI** is **NOT** installed
>    > - In your exercise, this section may appear disabled/commented (this is just to reduce time for the exercise, but in the real Dockerfile it is fully functional)
>    >
>    > <p>&nbsp;</p>
>    >
>    > ~~~
>    > #...........
>    > #Step 2. Download
>    > #Change to the installation dir, clone OpenFOAM directories
>    > ARG OFVERSIONGIT=$OFVERSION
>    > WORKDIR $OFINSTDIR
>    > 
>    > RUN git clone git://github.com/OpenFOAM/OpenFOAM-${OFVERSIONGIT}.git \
>    >  && git clone git://github.com/OpenFOAM/ThirdParty-${OFVERSIONGIT}.git
>    > ~~~
>    > {: .bash}
>    > - Cloning the source directories for OpenFOAM installation
>    {: .solution}
>
> 3. Now lets use the first Dockerfile to build a first Docker container (not the final main installation)
>
>    > ~~~
>    > ubuntu@vm:01_Docker$ docker build -f Dockerfile.01 -t myuser/openfoam:2.4.x.01 .
>    > ~~~
>    > {: .bash}
>    > - With the `-f` option you can choose a specific Dockerfile
>    > - **Important:** Do not forget the dot `.` (which means: current directory) at the end
>    > - Note the tag used: `2.4.x.01` as we know this is a first test and not the final image
>    >
>    > <p>&nbsp;</p>
>    >
>    > ~~~
>    > Sending build context to Docker daemon   21.5kB^M^M
>    > Step 1/15 : FROM pawsey/mpich-base:3.1.4_ubuntu16.04
>    >  ---> b2cb97823381
>    > Step 2/15 : LABEL maintainer="Alexis.Espinosa@pawsey.org.au"
>    >  ---> Running in a4f596b9337f
>    > Removing intermediate container a4f596b9337f
>    >  ---> a58d84a78040
>    > Step 3/15 : ARG OFVERSION="2.4.x"
>    >  ---> Running in 453210a478c6
>    > Removing intermediate container 453210a478c6
>    >  ---> 6d3d0e059b3a
>    > Step 4/15 : SHELL ["/bin/bash", "-c"]
>    > .
>    > .
>    > .
>    > Step 13/15 : ARG OFVERSIONGIT=$OFVERSION
>    >  ---> Running in 0f20db6d29f3
>    > Removing intermediate container 0f20db6d29f3
>    >  ---> 46ee1dca0673
>    > Step 14/15 : WORKDIR $OFINSTDIR
>    >  ---> Running in 40ecd1e76151
>    > Removing intermediate container 40ecd1e76151
>    >  ---> 0432945652d9
>    > Step 15/15 : RUN git clone git://github.com/OpenFOAM/OpenFOAM-${OFVERSIONGIT}.git  && git clone git://github.com/OpenFOAM/ThirdParty-${OFVERSIONGIT}.git
>    >  ---> Running in 5fdd676ad176
>    > ^[[91mCloning into 'OpenFOAM-2.4.x'...
>    > ^[[0m^[[91mCloning into 'ThirdParty-2.4.x'...
>    > ^[[0mRemoving intermediate container 5fdd676ad176
>    >  ---> bdbe2e3bac9e
>    > Successfully built bdbe2e3bac9e
>    > Successfully tagged myuser/openfoam:2.4.x.01
>    > ~~~
>    > {: .output}
>    >
>
> 4. Check that the image exists:
>    >
>    > ~~~
>    > ubuntu@vm:01_Docker$ docker image ls
>    > ~~~
>    > {: .bash}
>    >
>    > ~~~
>    > REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
>    > myuser/openfoam     2.4.x.01            bdbe2e3bac9e        49 minutes ago      866MB
>    > pawsey/openfoam     2.4.x               4917982ed825        2 weeks ago         7.42GB
>    > pawsey/openfoam     v1912               c4360b0a0f3f        2 weeks ago         11.9GB
>    > pawsey/mpich-base   3.1.4_ubuntu16.04   b2cb97823381        3 weeks ago         548MB
>    > ~~~
>    > {: .output}
>    
{: .discussion}

<p>&nbsp;</p>

## E. Updating the OpenFOAM settings files

- For the OpenFOAM installation to understand the use of the system-MPI (MPICH in this case), we need to modify the default settings

- We also need to modify the indication for the path of installation

- And the indication for the path where user's solvers and development will be stored

- All those setting need to be indicated in the files `bashrc` and `prefs.sh`

<p>&nbsp;</p>

Let's take a look to the second Dockerfile (`Dockerfile.02`)

> ## The `Dockerfile.02` file (main sections for discussion)
>
> ~~~
> #...........
> #Step 3. Definitions for the prefs and bashrc files.
> ARG OFPREFS=${OFINSTDIR}/OpenFOAM-${OFVERSION}/etc/prefs.sh
> ARG OFBASHRC=${OFINSTDIR}/OpenFOAM-${OFVERSION}/etc/bashrc
> ~~~
> {: .bash}
> - We'll make use of these variables (arguments) to refer to the `prefs.sh` and `bashrc` OpenFOAM settings files
>
> <p>&nbsp;</p>
>
> ~~~
> #...........
> #Defining the prefs.sh:
> RUN head -23 ${OFINSTDIR}/OpenFOAM-${OFVERSION}/etc/config/example/prefs.sh > $OFPREFS \
>  && echo '#------------------------------------------------------------------------------' >> ${OFPREFS} \
> #Using a combination of the variable definition recommended for the use of system mpich in this link:
> #   https://bugs.openfoam.org/view.php?id=1167
> #And in the file .../OpenFOAM-$OFVERSION/wmake/rules/General/mplibMPICH
> #(These MPI_* environmental variables are set in the prefs.sh,
> # and this file will be sourced automatically by the bashrc when the bashrc is sourced)
> #
> #--As suggested in the link above and mplibMPICH file:
>  && echo 'export WM_MPLIB=SYSTEMMPI' >> ${OFPREFS} \
>  && echo 'export MPI_ROOT="/usr"' >> ${OFPREFS} \
>  && echo 'export MPI_ARCH_FLAGS="-DMPICH_SKIP_MPICXX"' >> ${OFPREFS} \
>  && echo 'export MPI_ARCH_INC="-I${MPI_ROOT}/include"' >> ${OFPREFS} \
>  && echo 'export MPI_ARCH_LIBS="-L${MPI_ROOT}/lib${WM_COMPILER_LIB_ARCH} -L${MPI_ROOT}/lib -lmpich -lrt"' >> ${OFPREFS} \
>  && echo ''
> ~~~
> {: .bash}
> - Using echo command to write the settings into the `prefs.sh` file
>
> <p>&nbsp;</p>
>
> ~~~
> #...........
> #Modifying the bashrc file
> RUN cp ${OFBASHRC} ${OFBASHRC}.original \
> #Changing the installation directory within the bashrc file (This is not in the openfoamwiki instructions)
>  && sed -i '/^foamInstall=$HOME.*/afoamInstall='"${OFINSTDIR}" ${OFBASHRC} \
>  && sed -i '0,/^foamInstall=$HOME/s//# foamInstall=$HOME/' ${OFBASHRC} \
> #Changing the place for your own tools/solvers (WM_PROJECT_USER_DIR directory) within the bashrc file 
>  && sed -i '/^export WM_PROJECT_USER_DIR=.*/aexport WM_PROJECT_USER_DIR="'"${OFUSERDIR}/ofuser"'-$WM_PROJECT_VERSION"' ${OFBASHRC} \
>  && sed -i '0,/^export WM_PROJECT_USER_DIR/s//# export WM_PROJECT_USER_DIR/' ${OFBASHRC} \
>  && echo ''
> ~~~
> {: .bash}
> - Using `sed` command to replace settings in the `bashrc` file
>
> <p>&nbsp;</p>
>
{: .solution}

## F. Building a second image 

In this second exercise, the building process will advance a bit further, but will stop at the point where the following trick is set:

> ## The `RUN StopHere` trick in the `Dockerfile.02`:
> ~~~
> #...........
> # REMOVE THIS AFTER TESTING
> RUN StopHere #Trick for stopping the recipe at this point
> #...........
> ~~~
> {: .bash}
> - We can use this trick to stop the building process at some point and check the installation up to there
> - I like to use this trick instead of commenting the rest of the Dockerfile
> - Building process will end with an error, but previous commands are cached in layers
> - We can still access any cached layer as an image
>
{: .solution}

> ## Steps for the second build:
>
> 1. Build the docker container using `Dockerfile.02` 
>    > ~~~
>    > ubuntu@vm:01_Docker$ docker build -f Dockerfile.02 -t myuser/openfoam:2.4.x.02 .
>    > ~~~
>    > {: .bash}
>    > 
>    > ~~~
>    > Sending build context to Docker daemon     64kB^M^M
>    > Step 1/39 : FROM pawsey/mpich-base:3.1.4_ubuntu16.04
>    >  ---> b2cb97823381
>    > Step 2/39 : LABEL maintainer="Alexis.Espinosa@pawsey.org.au"
>    >  ---> Using cache
>    >  ---> a58d84a78040
>    > Step 3/39 : ARG OFVERSION="2.4.x"
>    >  ---> Using cache
>    >  ---> 6d3d0e059b3a
>    > Step 4/39 : SHELL ["/bin/bash", "-c"]
>    >  ---> Using cache
>    >  ---> 9812983b6344
>    > .
>    > .
>    > .
>    > Removing intermediate container 5e181146e2bd
>    >  ---> 46e4f0fcea46
>    > Step 19/39 : RUN cp ${OFBASHRC} ${OFBASHRC}.original  && sed -i '/^foamInstall=$HOME.*/afoamInstall='"${OFINSTDIR}" ${OFBASHRC}  && sed -i '0,/^foamInstall=$HOME/s//# foamInstall=$HOME/' ${OFBASHRC}  && sed -i '/^export WM_PROJECT_USER_DIR=.*/aexport WM_PROJECT_USER_DIR="'"${OFUSERDIR}/ofuser"'-$WM_PROJECT_VERSION"' ${OFBASHRC}  && sed -i '0,/^export WM_PROJECT_USER_DIR/s//# export WM_PROJECT_USER_DIR/' ${OFBASHRC}  && echo ''
>    >  ---> Running in 569bea603d57
>    > 
>    > Removing intermediate container 569bea603d57
>    >  ---> e4ae93bc92f1
>    > Step 20/39 : RUN StopHere #Trick for stopping the recipe at this point
>    >  ---> Running in 35a6be1d6690
>    > ^[[91m/bin/bash: StopHere: command not found
>    > ^[[0mThe command '/bin/bash -c StopHere #Trick for stopping the recipe at this point' returned a non-zero code: 127
>    > ~~~
>    > {: .output}
>    > - Everything went fine, except for `Step 20/39 : RUN StopHere`
>    > - Exactly above that line there is a hexadecimal number, in this case: `---> e4ae93bc92f1`
>    > - That number is the "ID" of the cache layer before the error, and we can access it as an image
>    > 
>    >
> 2. Use the ID of the latest cached layer (copy/paste) for running the image interactively.
>    > ~~~
>    > ubuntu@vm:01_Docker$ docker run -it --rm e4ae93bc92f1
>    > ~~~
>    > {: .bash}
>    > - The use of an interactive session for this kind of checks is very practical
>    > - The `-it` indicates interactive
>    > - The `--rm` indicates docker to remove the running session (container) from the docker engine after exiting
>    > - (`--rm` removes the container, but not the image. The image is safe)
>    > 
>    > ~~~
>    > root@202a0bf870bb:/opt/OpenFOAM#
>    > ~~~
>    > {: .output}
>    > - We are now in an interactive session inside the container
>
> 3. Inside the container, check the contents of the `prefs.sh` file: 
>
>    > ~~~
>    > root@202a0bf870bb:/opt/OpenFOAM# cd OpenFOAM-2.4.x/etc
>    > root@202a0bf870bb:/opt/OpenFOAM/OpenFOAM-2.4.x/etc# cat prefs.sh
>    > ~~~
>    > {: .bash}
>    > 
>    > ~~~
>    > #----------------------------------*-sh-*--------------------------------------
>    > # =========                 |
>    > # \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox
>    > #  \\    /   O peration     |
>    > #   \\  /    A nd           | Copyright (C) 2011-2013 OpenFOAM Foundation
>    > #    \\/     M anipulation  |
>    > #------------------------------------------------------------------------------
>    > # License
>    > #     This file is part of OpenFOAM.
>    > #
>    > #     OpenFOAM is free software: you can redistribute it and/or modify it
>    > #     under the terms of the GNU General Public License as published by
>    > #     the Free Software Foundation, either version 3 of the License, or
>    > #     (at your option) any later version.
>    > #
>    > #     OpenFOAM is distributed in the hope that it will be useful, but WITHOUT
>    > #     ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
>    > #     FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
>    > #     for more details.
>    > #
>    > #     You should have received a copy of the GNU General Public License
>    > #     along with OpenFOAM.  If not, see <http://www.gnu.org/licenses/>.
>    > #
>    > #------------------------------------------------------------------------------
>    > export WM_MPLIB=SYSTEMMPI
>    > export MPI_ROOT="/usr"
>    > export MPI_ARCH_FLAGS="-DMPICH_SKIP_MPICXX"
>    > export MPI_ARCH_INC="-I${MPI_ROOT}/include"
>    > export MPI_ARCH_LIBS="-L${MPI_ROOT}/lib${WM_COMPILER_LIB_ARCH} -L${MPI_ROOT}/lib -lmpich -lrt"
>    > ~~~
>    > {: .output}
>    > - Settings are as expected
>
> 4. Also check the modifications inside the `bashrc` file 
>    >
>    > ~~~
>    > root@202a0bf870bb:/opt/OpenFOAM/OpenFOAM-2.4.x/etc# grep "foamInstall" bashrc
>    > ~~~
>    > {: .bash}
>    > 
>    > ~~~
>    > # 'foamInstall' below to where OpenFOAM is installed
>    > # foamInstall=$HOME/$WM_PROJECT
>    > foamInstall=/opt/OpenFOAM
>    > # foamInstall=~$WM_PROJECT
>    > # foamInstall=/opt/$WM_PROJECT
>    > # foamInstall=/usr/local/$WM_PROJECT
>    > : ${FOAM_INST_DIR:=$foamInstall}; export FOAM_INST_DIR
>    > unset cleaned foamClean foamInstall foamOldDirs
>    > ~~~
>    > {: .output}
>    > - Setting is correct
>    > 
>    > ~~~
>    > root@202a0bf870bb:/opt/OpenFOAM/OpenFOAM-2.4.x/etc# grep "PROJECT_USER_DIR" bashrc
>    > ~~~
>    > {: .bash}
>    > 
>    > ~~~
>    > # export WM_PROJECT_USER_DIR=$HOME/$WM_PROJECT/$USER-$WM_PROJECT_VERSION
>    > export WM_PROJECT_USER_DIR="/home/ofuser/OpenFOAM/ofuser-$WM_PROJECT_VERSION"
>    > ~~~
>    > {: .output}
>    > - Setting is correct
>
> 5. Exit the interactive session
>    >
>    > ~~~
>    > root@202a0bf870bb:/opt/OpenFOAM/OpenFOAM-2.4.x/etc# exit
>    > ~~~
>    > {: .bash}
>    > 
{: .challenge}

## G. Final remarks on Docker

- The official maintained Dockerfile for OpenFOAM-2.4.x is in the PawseySC GitHUB:
[https://github.com/PawseySC/pawsey-containers](https://github.com/PawseySC/pawsey-containers)

- Use existing Dockerfiles (and instructions from the Developers and OpenFOAM wiki: [https://openfoamwiki.net/index.php/Category:Installing_OpenFOAM_on_Linux](https://openfoamwiki.net/index.php/Category:Installing_OpenFOAM_on_Linux)) to build your own container 

> ## The `Dockerfile` file (some other sections for discussion)
>
> ~~~
> #Third party compilation
> RUN . ${OFBASHRC} \
>  && cd $WM_THIRD_PARTY_DIR \
>  && ./Allwmake 2>&1 | tee log.Allwmake
> ~~~
> {: .bash}
>
> ~~~
> #OpenFOAM compilation 
> ENV WM_NCOMPPROCS=4
> RUN . ${OFBASHRC} \
>  && cd $WM_PROJECT_DIR \
>  && export QT_SELECT=qt4 \
>  && ./Allwmake 2>&1 | tee log.Allwmake
> ~~~
> {: .bash}
> - Always source the `bashrc` file (in this case using the argument `OFBASHRC`) before compilation steps in a layer
> - This is because environment settings are lost after each `RUN` command
> - The only way to keep the environment variables is to apply a `ENV` command for each of the OpenFOAM variables, which is a very tedious process
> - When executing the Docker container, the `bashrc` file will need to be sourced too
>
{: .callout}

> ## Final steps in the creation of the full Docker container
>
> 1. The final building step (after reaching the right recipe) is:
>
>    ~~~
>    ubuntu@vm:01_Docker$ docker build -t myuser/openfoam:2.4.x .
>    ~~~
>    {: .bash}
>    - **(This will not work in the current state of the exercise)**, but still its good to have the instruction here
>    - Without the `-f` option, then the `Dockerfile` will be used by default
>    - The full built of the `pawsey/openfoam:2.4.x` took around 6 hours
>   
>    ~~~
>    Sending build context to Docker daemon  16.38kB^M^M
>    Step 1/38 : FROM pawsey/mpich-base:3.1.4_ubuntu16.04
>     ---> b2cb97823381
>    Step 2/38 : LABEL maintainer="Alexis.Espinosa@pawsey.org.au"
>     ---> Using cache
>     ---> 2d15a7d6c656
>    Step 3/38 : ARG OFVERSION="2.4.x"
>     ---> Running in 0126259b03e1
>    .
>    .
>    .
>    Removing intermediate container 7a3762b35d26
>     ---> 9e966f97d1fb
>    Step 37/38 : USER ofuser
>     ---> Running in 11bbf508ba96
>    Removing intermediate container 11bbf508ba96
>     ---> 16869a33968e
>    Step 38/38 : WORKDIR /home/ofuser
>     ---> Running in 121eb0c2151e
>    Removing intermediate container 121eb0c2151e
>     ---> 4917982ed825
>    Successfully built 4917982ed825
>    Successfully tagged myuser/openfoam:2.4.x
>    ~~~
>    {: .output}
>    
> 2. Then we test that the internal OpenFOAM installation works properly
>    - This can easily be checked with an interactive session
>    - **(Here we test with the `pawsey/openfoam:2.4.x` as the myuser container does not exist yet)**
>    
>    ~~~
>    ubuntu@vm:01_Docker$ docker run -it --rm pawsey/openfoam:2.4.x
>    ofuser@49146943821c:~$ source /opt/OpenFOAM/OpenFOAM-2.4.x/etc/bashrc
>    ~~~
>    {: .bash}
>    - You need to source the `bashrc` file to activate the OpenFOAM environment
>
>    ~~~
>    ofuser@49146943821c:~$ cd $FOAM_TUTORIALS
>    ofuser@49146943821c:tutorials$ ls
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Allclean  DNS         compressible      financial       lagrangian  resources
>    Allrun    basic       discreteMethods   heatTransfer    mesh        stressAnalysis
>    Alltest   combustion  electromagnetics  incompressible  multiphase
>    ~~~
>    {: .output}
>    
>    ~~~
>    ofuser@49146943821c:tutorials$ cd incompressible/pimpleFoam/channel395
>    ofuser@49146943821c:channel395$ ./Allrun
>    ~~~
>    {: .bash}
>    - `./Allrun` scripts executes all the workflow for the tutorial
>    - Usually you do not want to execute the tutorials from the original directory (because you can modify your original source)
>    - But in this case it is fine because changes will be lost when exiting the container
>    - If you want to keep the results, you would need to extract the tutorial into a local directory first
>    
>    ~~~
>    Running blockMesh on /opt/OpenFOAM/OpenFOAM-2.4.x/tutorials/incompressible/pimpleFoam/channel395
>    Running decomposePar on /opt/OpenFOAM/OpenFOAM-2.4.x/tutorials/incompressible/pimpleFoam/channel395
>    Running pimpleFoam in parallel on /opt/OpenFOAM/OpenFOAM-2.4.x/tutorials/incompressible/pimpleFoam/channel395 using 5 processes 
>    ~~~
>    {: .output}
>    - Everything looks fine.
>    - This may take several minutes. Use `<Ctrl-C>` to kill the solver
>    
>    ~~~
>    ofuser@49146943821c:channel395$ exit
>    ~~~
>    {: .bash}
>
> 3. Finally, push your container to DockerHub. **(myuser needs an account in DockerHub first)**
>
>    ~~~
>    ubuntu@vm:01_Docker$ docker push myuser/openfoam:2.4.x
>    ~~~
>    {: .bash}
>    
>    ~~~
>    The push refers to repository [docker.io/myuser/openfoam]
>    6b38074bb2ff: Preparing 
>    30bc9c9ededb: Preparing 
>    a238555104b7: Preparing 
>    30bc9c9ededb: Mounted from pawsey/openfoam 
>    c49ae6578794: Mounted from pawsey/openfoam 
>    45cc33e9b7df: Waiting 
>    502628a9589d: Waiting
>    .
>    .
>    .
>    e79142719515: Layer already exists 
>    aeda103e78c9: Layer already exists 
>    2558e637fbff: Layer already exists 
>    f749b9b0fb21: Layer already exists 
>    2.4.x: digest: sha256:717d3e4153c52b517273e2afaadf2e38651c7ecf1d68ce09658fc68e2df806a0 size: 7227
>    ~~~
>    {: .output}
>    - Again, this may not work at this stage of the exercise because the full image has not been created
>    - You would also need a DockerHub account to be allowed to push images
>
{: .callout}

## H. Converting the container into Singularity format

First, lets take a look into the `Singularity.def` definition file:

> ## The `Singularity.def` file:
>
> ~~~
> Bootstrap: docker
> From: pawsey/openfoam:2.4.x
> 
> %post
> /bin/mv /bin/sh /bin/sh.original
> /bin/ln -s /bin/bash /bin/sh
> echo ". /opt/OpenFOAM/OpenFOAM-2.4.x/etc/bashrc" >> $SINGULARITY_ENVIRONMENT
> ~~~
> {: .bash}
> - This is the whole file! Very simple!
> - The singularity image will be built from the Pawsey one
> - The first two lines in the `%post` section instructs the builder to use `bash` as the shell
> - This is needed because OpenFOAM scripts have some instructions that can only be interpreted by `bash` (called bash-isms by geeks)
> - The last line writes the instruction of sourcing the bashrc file every time the container is used
{: .solution}
>
> ## Steps for porting the image into Singularity:
> 1. cd into the `02_PortingToSingularity` directory
>    ~~~
>    ubuntu@vm:*-2.4.x$ cd 02_PortingToSingularity
>    ubuntu@vm:02_*Singularity$ ls
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Singularity.def
>    ~~~
>    {: .output}
>    
> 2. Use the singularity build:
>    
>    ~~~
>    ubuntu@vm:02_*Singularity$ sudo singularity build openfoam-2.4.x-myuser.sif Singularity.def
>    ~~~
>    {: .bash}
>    - You must be have sudo/root privileges to execute `build`
>    
>    ~~~
>    INFO:    Starting build...
>    Getting image source signatures
>    Copying blob f7277927d38a done
>    Copying blob 8d3eac894db4 done
>    Copying blob edf72af6d627 done
>    Copying blob 3e4f86211d23 done
>    Copying blob e69ac5970cdd done
>    Copying blob af93086ab594 done
>    Copying blob 9121f6d149f9 done
>    .
>    .
>    .
>    2020/05/27 01:34:22  info unpack layer: sha256:11c314071f65ec67414a027ec31d85459cf15f93be7d5d5945d54e9e2cf765c3
>    2020/05/27 01:34:22  info unpack layer: sha256:a5ff4ed9ae6c32eee3053982461f781edb56b0427a551c8b9541bccc0541234d
>    2020/05/27 01:34:22  info unpack layer: sha256:29d7c1e14df37b4b7c80226021b633150801041cc9c4401f7d35449608f3d9cf
>    2020/05/27 01:34:22  info unpack layer: sha256:06d6bd89758266248c1b8cf78a2d21a3fbcef805820f7b91d78ebf1b88d61621
>    2020/05/27 01:34:47  info unpack layer: sha256:3a96376985c106d721cc9d686d10acaad6084f535e326e4c4799149cccfaa39e
>    INFO:    Running post scriptlet
>    + /bin/mv /bin/sh /bin/sh.original
>    + /bin/ln -s /bin/bash /bin/sh
>    + echo . /opt/OpenFOAM/OpenFOAM-2.4.x/etc/bashrc
>    INFO:    Creating SIF file...
>    INFO:    Build complete: openfoam-2.4.x-myuser.sif
>    ~~~
>    {: .output}
>    
>    ~~~
>    ubuntu@vm:02_*Singularity$ ls
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Singularity.def  openfoam-2.4.x-myuser.sif
>    ~~~
>    {: .bash}
>    
> 3. Test a solver
>    
>    ~~~
>    ubuntu@vm:02_*Singularity$ mkdir run
>    ubuntu@vm:02_*Singularity$ singularity run -B ./run:/home/ofuser-2.4.x/run openfoam-2.4.x-myuser.sif
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Singularity> cd run
>    Singularity> cp -r /opt/OpenFOAM/OpenFOAM-2.4.x/tutorials/incompressible/pimpleFoam/channel395 .
>    Singularity> cd channel395
>    Singularity> ls
>    ~~~
>    {: .bash}
>
>    ~~~
>    0  0.org  Allrun  constant  system
>    ~~~
>    {: .output}
>
>    ~~~
>    Singularity> ./Allrun 
>    ~~~
>    {: .bash}
>
>    ~~~
>    Running blockMesh on /home/ubuntu/pawseyTraining/containers-openfoam-workshop-scripts/04_buildingAnOpenFOAMContainer/openfoam-2.4.x/02_PortingToSingularity/run/channel395
>    Running decomposePar on /home/ubuntu/pawseyTraining/containers-openfoam-workshop-scripts/04_buildingAnOpenFOAMContainer/openfoam-2.4.x/02_PortingToSingularity/run/channel395
>    Running pimpleFoam in parallel on /home/ubuntu/pawseyTraining/containers-openfoam-workshop-scripts/04_buildingAnOpenFOAMContainer/openfoam-2.4.x/02_PortingToSingularity/run/channel395 using 5 processes
>    ~~~
>    {: .output}
>    - The solver may take several minutes to finish.
>    - Use `<Ctrl-c>` to kill the solver
>
>    ~~~
>    Singularity> exit
>    ~~~
>    {: .bash}
>
>    ~~~
>    ubuntu@vm:02_*Singularity$ cd run/channel395/
>    ubuntu@vm:02_*Singularity$ ls
>    ~~~
>    {: .bash}
>
>    ~~~
>    0      Allrun    log.blockMesh     log.pimpleFoam   log.reconstructPar  processor1  processor3  system
>    0.org  constant  log.decomposePar  log.postChannel  processor0          processor2  processor4
>    ~~~
>    {: .output}
>    
> 4. Move the image to any system where you want to use it
>
{: .discussion} 
