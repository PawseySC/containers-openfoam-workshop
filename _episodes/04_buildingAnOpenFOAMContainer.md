---
title: "Building an OpenFOAM container with MPICH"
teaching: 30
exercises: 30
questions:
- How can I build my own OpenFOAM container with MPICH?
objectives:
- Explain our current recommended process for building OpenFOAM containers
keypoints:
- Take it easy, be patient
- Use existing definition files examples and available guides to define the right installation recipe
- Main difference with a standard OpenFOAM installation guide are':'
- 1.Avoiding any step that performs an installation of OpenMPI
- 2.Settings in `prefs.sh` for using `WM_MPLIB=SYSTEMMPI` (MPICH in this case) 
- 3.Settings in `bashrc` for defining the new location for installation
- 4.Settings in `bashrc` for defining `WM_PROJECT_USER_DIR`
---


## 0. Introduction

> ## Why you need to build your own container?
>
> - If the version of OpenFOAM that you need to use is not maintained by Pawsey
> - If you want to move your OpenFOAM "installation" easily to another system
> - If you want to use the OverlayFS approach to reduce the amount of reduced files (to be explained in the following episode)
>
{: .callout}

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

- If not, for the workshops we provide a Nimbus Virtual Machines attendees:

> ## A.I Steps for connecting to a Nimbus Virtual Machine for a live workshop:
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
{: .solution}

<p>&nbsp;</p>

## B. Clone the Git repository for the workshop exercises

You'll need to clone the git repository into your linux environment:

> ## B.I Steps for cloning the Git repository
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
{: .discussion}

<p>&nbsp;</p>

## C. Building a first Docker image with MPICH

> ## General advice:
> - As mentioned in the previous seminars, builiding containers of large applications will need some trial-and-error process
> 
> - The building of an OpenFOAM image needs trial-and-error. (Installations of OpenFOAM are sometimes very time consuming)
> 
> - Therefore, we recommended to build the image first with Docker and convert it to Singularity afterwards
> 
> - This also allows more portability
>
{: .callout}

> ## C.I Steps for building our first image based on `pawsey/mpich-base`
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
> 2. If you wish, shrink your prompt, execute this:
>
>    ~~~
>    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\W\[\033[00m\]\$ '
>    ~~~
>    {: .bash}
>
>    ~~~
>    a shrinked prompt $ 
>    ~~~
>    {: .output}
>
> 3. Let's take a look into the first Dockerfile (`Dockerfile.01`):
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
>    > {: .docker}
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
>    > {: .docker}
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
>    > {: .docker}
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
>    > {: .docker}
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
>    > {: .docker}
>    > - Cloning the source directories for OpenFOAM installation
>    {: .solution}
>
> 4. Now lets use the first Dockerfile to build a first Docker container (not the final main installation)
>
>    > ~~~
>    > ubuntu@vm:01_Docker$ sudo docker build -f Dockerfile.01 -t myuser/openfoam:2.4.x.01 .
>    > ~~~
>    > {: .bash}
>    > - With the `-f` option you can choose a specific Dockerfile
>    > - **Important:** Do not forget the dot `.` (which means: current directory) at the end
>    > - Note the tag used: `2.4.x.01` as we know this is a first test and not the final image
>    > - **Important:** The `myuser` name is important. This should be modified to correspond to your **Dockerhub** account. And will be used when the final version of the image is ready to be pushed it into DockerHub
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
> 5. Check that the image exists:
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
>    - Our image is now first in the list
>    - (For those using the virtual machines provided by Pawsey to the atendees, note that some pawsey images were already pulled before hand)
>    
{: .discussion}

<p>&nbsp;</p>

## D. More on the Dockerfile

> ## Non-default settings in the installation are:
> - For the OpenFOAM installation to understand the use of the system-MPI (MPICH in this case), we need to modify the default settings
> 
> - We also recommend to modify the path for the OpenFOAM installation
> 
> - We also recommend the path where the user's own solvers and libraries will be stored (`WM_PROJECT_USER_DIR`)
> 
> - These three modifications are set in the files `bashrc` and `prefs.sh`
> 
{: .callout}

> ## The `Dockerfile.02` file (main parts for discussion)
>
> ~~~
> #...........
> #Step 3. Definitions for the prefs and bashrc files.
> ARG OFPREFS=${OFINSTDIR}/OpenFOAM-${OFVERSION}/etc/prefs.sh
> ARG OFBASHRC=${OFINSTDIR}/OpenFOAM-${OFVERSION}/etc/bashrc
> ~~~
> {: .docker}
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
> {: .docker}
> - These are the settings for the use of the system installed MPICH
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
> {: .docker}
> - These are the settings for the installation path and the `WM_PROJECT_USER_DIR` path
> - Using `sed` command to replace settings in the `bashrc` file
> - The syntax shown here adds the new setting immediately below the original setting
> - Then comments out the line of the original setting
> - Note also that the original file is first copied into `bashrc.original`
>
> <p>&nbsp;</p>
>
{: .solution}

> ## The `RUN StopHere` trick in the `Dockerfile.02`:
> ~~~
> #...........
> # REMOVE THIS AFTER TESTING
> RUN StopHere #Trick for stopping the recipe at this point
> #...........
> ~~~
> {: .docker}
> - We recommend the use of this trick to stop the building process at some point and check the status of the installation up to that point
> - We like to use this trick instead of commenting the rest of the Dockerfile
> - Note that the building process will end with an error, but previous commands are cached in layers
> - We can still access any succesful cached layer before the error and use it as an image
>
{: .callout}

> ## D.I Steps for the second build:
>
> 1. Build the docker container using `Dockerfile.02` 
>    > ~~~
>    > ubuntu@vm:01_Docker$ sudo docker build -f Dockerfile.02 -t myuser/openfoam:2.4.x.02 .
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
> 2. Use the ID of the latest cached layer (copy/paste) for running that layer as an image interactively.
>    > ~~~
>    > ubuntu@vm:01_Docker$ docker run -it --rm e4ae93bc92f1
>    > ~~~
>    > {: .bash}
>    > - The use of an interactive session for this kind of checks is very practical
>    > - The `-it` indicates interactive
>    > - The `--rm` indicates docker to remove the running session (container) from the docker engine after exiting
>    > - (After exiting `--rm` removes the container, but not the image. The image is safe)
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
>    > root@202a0bf870bb:/opt/OpenFOAM# pwd
>    > ~~~
>    > {: .bash}
>    > 
>    > ~~~
>    > /opt/OpenFOAM
>    > ~~~
>    > {: .output}
>    > 
>    > ~~~
>    > root@202a0bf870bb:/opt/OpenFOAM# ls
>    > ~~~
>    > {: .bash}
>    > 
>    > ~~~
>    > OpenFOAM-2.4.x  ThirdParty-2.4.x
>    > ~~~
>    > {: .output}
>    > 
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
>    > - The only real active definition of `foamInstall` is correct, the rest are commented
>    > 
>    > <p>&nbsp;</p>
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
>    > - The setting of `WM_PROJECT_USER_DIR` is correct
>
> 5. Exit the interactive session
>    >
>    > ~~~
>    > root@202a0bf870bb:/opt/OpenFOAM/OpenFOAM-2.4.x/etc# exit
>    > ~~~
>    > {: .bash}
>    > 
{: .challenge}

<p>&nbsp;</p>

## E. Final remarks on the building of OpenFOAM containers at Pawsey with Docker

> ## General recommendations:
> - Pawsey maintains several images that can be used for running applications or for building new ones
> 
> - The official maintained Dockerfile for OpenFOAM-2.4.x is in the PawseySC GitHUB: [https://github.com/PawseySC/pawsey-containers](https://github.com/PawseySC/pawsey-containers)
> 
> - The DockerHub repository (where the corresponding Docker images are) is: [https://hub.docker.com/u/pawsey](https://hub.docker.com/u/pawsey)
> 
> - Use pawsey Dockerfiles as examples to build your own container 
> 
> - Then write your own Dockerfile and modify it following the instructions from the Developers and from the OpenFOAM wiki: [https://openfoamwiki.net/index.php/Category:Installing_OpenFOAM_on_Linux](https://openfoamwiki.net/index.php/Category:Installing_OpenFOAM_on_Linux))
> 
> - The process may need some trial and error until you reach the correct recipe (the use of the `StopHere` trick shown in the previous exercise may help to debug the process and save some time)
> 
> - After reaching a final correct Dockerfile recipe you can then build your Docker image and convert it to Singularity in a second stage
>
{: .prereq}


> ## Compilation sections in the `Dockerfile` and the `bashrc` file
>
> ~~~
> #Third party compilation
> RUN . ${OFBASHRC} \
>  && cd $WM_THIRD_PARTY_DIR \
>  && ./Allwmake 2>&1 | tee log.Allwmake
> ~~~
> {: .docker}
>
> <p>&nbsp;</p>
>
> ~~~
> #OpenFOAM compilation 
> ENV WM_NCOMPPROCS=4
> RUN . ${OFBASHRC} \
>  && cd $WM_PROJECT_DIR \
>  && export QT_SELECT=qt4 \
>  && ./Allwmake 2>&1 | tee log.Allwmake
> ~~~
> {: .docker}
> - In each `RUN` command, source the `bashrc` file (in this case using the argument `OFBASHRC`) before compilation steps in a layer
> - This is because environment settings are lost after each `RUN` command
> - The only way to keep the environment variables is to apply a `ENV` command for each of the OpenFOAM variables, which is a very tedious process and we do not recommend it
> - Instead, when executing the Docker container, the `bashrc` file will need to be sourced to set the OpenFOAM environment
> - Nevertheless, we are recommending to use Singularity to run your OpenFOAM containers, so it is possible that you will not be using the Docker image except for testing
> - Fortunately, when builiding the Singularity image, there is an easy way to indicate to source the `bashrc` file every time the container is executed
> - Then, for Singularity containers, there will be no need to manually source the `bashrc` file
>
{: .callout}

> ## E.I Final steps in the creation of the full Docker container
>
> 1. The final building step (after reaching the right `Dockerfile` recipe) would be:
>
>    ~~~
>    ubuntu@vm:01_Docker$ sudo docker build -t myuser/openfoam:2.4.x .
>    ~~~
>    {: .bash}
>    - Without the `-f` option, then a file named `Dockerfile` will be used by default as the recipe
>    - **(This command will not work in the current state of the exercise)** because that file has not been created yet
>    - But still we are presenting the instruction here for completeness
>    - To really build the full image you would need to create the correct `Dockerfile` (the provided file `*.02` was produced from the official recipe, but it has been modified by commenting some lines (marked with `#commentedForExercise#` string) and the additional line with `RUN StopHere` 
>    - To create your own correct `Dockerfile` you can remove the comments and the `StopHere` line, or clone the git repository mentioned a few paragraphs above and take the correct recipe from there
>    - The full building process of the `openfoam:2.4.x` takes around 6 hours
>
>    <p>&nbsp;</p>
>   
>    - When using the correct `Dockerfile` the output will like like this:
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
>    - The name `myuser` is important and should be modified to match your username in DockerHub
>    - This because it will indicate the owner when pushing the container image
>
>    <p>&nbsp;</p>
>    
> 2. Then we test that the internal OpenFOAM installation works properly
>    - This can easily be checked with an interactive session
>    - **(Here we perform the test with the `pawsey/openfoam:2.4.x` image because the `myuser/openfoam:2.4.x` image does not exist yet):**
>    
>    ~~~
>    ubuntu@vm:01_Docker$ docker run -it --rm pawsey/openfoam:2.4.x
>    ofuser@49146943821c:~$ source /opt/OpenFOAM/OpenFOAM-2.4.x/etc/bashrc
>    ~~~
>    {: .bash}
>    - This will also show the way to run the Docker container in an interactive session
>    - To set the OpenFOAM environment in our container, the user needs to source manually the `bashrc` file
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
>    - In a common installation of OpenFOAM, usually you do not want to execute the tutorials from the original directory (because you can modify your original source)
>    - But in this case, it is fine the original image will not be modified and the changes we observe here will be lost when exiting the container
>    - If you want to keep the results, you would need to extract the tutorial into a local host directory first and execute the containerised solver from there (mounting the local directory in the docker command) (not explained here)
>    - Or you could also copy out the results (also by mounting the local directory in the docker command, so you have a directory where to copy the results) (not explained here)
>    - (Refer to the Docker documentation if you want to use the Docker image in your computer or elsewhere)
>    
>    ~~~
>    Running blockMesh on /opt/OpenFOAM/OpenFOAM-2.4.x/tutorials/incompressible/pimpleFoam/channel395
>    Running decomposePar on /opt/OpenFOAM/OpenFOAM-2.4.x/tutorials/incompressible/pimpleFoam/channel395
>    Running pimpleFoam in parallel on /opt/OpenFOAM/OpenFOAM-2.4.x/tutorials/incompressible/pimpleFoam/channel395 using 5 processes 
>    ~~~
>    {: .output}
>    - OpenFOAM execution is working properly.
>    - This may take several minutes (~15 to ~30 min). Use `<Ctrl-C>` to kill the solver
>    
>    ~~~
>    ofuser@49146943821c:channel395$ exit
>    ~~~
>    {: .bash}
>
> 3. After testing that the Docker container works properly, the final step is to push your container to DockerHub.
>    - As mentioned some paragraphs above, **(the `myuser` name in the commands needs to be replaced with your real account in DockerHub)**
>    - (Refer to Docker documentation and website to obtain your DockerHub account) (not explained here)
>
>    - For example:
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
>
{: .discussion}

## F. Converting the container into Singularity format

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
> {: .singularity}
> - This is the whole definition file for the conversion to singularity. Very simple!
> - The singularity image will be built from the Pawsey one: `pawsey/openfoam:2.4.x`
> - The image in the `From:` command need to exist in the DockerHub registry (repository)
> - The first two lines in the `%post` section instructs the builder to use `bash` as the shell
> - This is needed because OpenFOAM scripts have some instructions that can only be interpreted by `bash` (called bash-isms by geeks)
> - The last line writes the instruction of sourcing the bashrc file every time the a container instance is created with this image
{: .callout}

> ## F.I Steps for porting the image into Singularity:
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
>    - You must have sudo/root privileges to execute `build`
>    - conversion may take around 10 minutes
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
{: .discussion}

> ## F.II Test the Singularity image
>
> 1. Run the container interactively and copy the tutorial to the host:
>    
>    ~~~
>    ubuntu@vm:02_*Singularity$ singularity run openfoam-2.4.x-myuser.sif
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Singularity> mkdir run
>    Singularity> cd run
>    Singularity> cp -r $FOAM_TUTORIALS/incompressible/pimpleFoam/channel395 .
>    Singularity> cd channel395
>    Singularity> ls
>    ~~~
>    {: .bash}
>    - Note that the directory from where singularity was called is mounted by default, so you can copy stuff from the interior of the container to the host file system
>
>    ~~~
>    0  0.org  Allrun  constant  system
>    ~~~
>    {: .output}
>
> 2. Use the OpenFOAM tools to if they work. First, create the mesh:
>    ~~~
>    Singularity> blockMesh
>    ~~~
>    {: .bash}
>
>    ~~~
>    /*---------------------------------------------------------------------------*\
>    | =========                 |                                                 |
>    | \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
>    |  \\    /   O peration     | Version:  2.4.x                                 |
>    |   \\  /    A nd           | Web:      www.OpenFOAM.org                      |
>    |    \\/     M anipulation  |                                                 |
>    \*---------------------------------------------------------------------------*/
>    Build  : 2.4.x-2b147f41daf9
>    Exec   : blockMesh
>    .
>    .
>    .
>    ----------------
>      patch 0 (start: 175300 size: 1200) name: bottomWall
>      patch 1 (start: 176500 size: 1200) name: topWall
>      patch 2 (start: 177700 size: 1000) name: sides1_half0
>      patch 3 (start: 178700 size: 1000) name: sides1_half1
>      patch 4 (start: 179700 size: 1000) name: sides2_half0
>      patch 5 (start: 180700 size: 1000) name: sides2_half1
>      patch 6 (start: 181700 size: 750) name: inout1_half0
>      patch 7 (start: 182450 size: 750) name: inout1_half1
>      patch 8 (start: 183200 size: 750) name: inout2_half0
>      patch 9 (start: 183950 size: 750) name: inout2_half1
>    
>    End
>    ~~~
>    {: .output}
>    
> 3. Run the decomposer
>
>    ~~~
>    Singularity> decomposePar
>    ~~~
>    {: .bash}
>    
>    ~~~
>    /*---------------------------------------------------------------------------*\
>    | =========                 |                                                 |
>    | \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
>    |  \\    /   O peration     | Version:  2.4.x                                 |
>    |   \\  /    A nd           | Web:      www.OpenFOAM.org                      |
>    |    \\/     M anipulation  |                                                 |
>    \*---------------------------------------------------------------------------*/
>    Build  : 2.4.x-2b147f41daf9
>    Exec   : decomposePar
>    .
>    .
>    .
>    Time = 0
>    
>    Processor 0: field transfer
>    Processor 1: field transfer
>    Processor 2: field transfer
>    Processor 3: field transfer
>    Processor 4: field transfer
>    
>    End.
>    ~~~
>    {: .output}
>    
> 4. Modify the `system/controlDict` to write results every time step (just for this particular test)
>    - set the line of writeInterval to read:
>    - `writeInterval  1;`
>    - You can edit the file, or simply use the following `sed` replacement:
>
>    ~~~
>    Singularity> sed -i 's,^writeInterval.*,writeInterval  1;,' system/controlDict 
>    ~~~
>    {: .bash}
>
>    - Check the setting:
>    ~~~
>    Singularity> grep "writeInterval" system/controlDict 
>    ~~~
>    {: .bash}
>
>    ~~~
>    writeInterval  1;
>    ~~~
>    {: .output}
>
> 5. Run the parallel solver
>
>    ~~~
>    Singularity> mpiexec -n 5 pimpleFoam -parallel
>    ~~~
>    {: .bash}
>    
>    ~~~
>    /*---------------------------------------------------------------------------*\
>    | =========                 |                                                 |
>    | \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
>    |  \\    /   O peration     | Version:  2.4.x                                 |
>    |   \\  /    A nd           | Web:      www.OpenFOAM.org                      |
>    |    \\/     M anipulation  |                                                 |
>    \*---------------------------------------------------------------------------*/
>    Build  : 2.4.x-2b147f41daf9
>    Exec   : pimpleFoam -parallel
>    .
>    .
>    .
>    Courant Number mean: 0.30276 max: 0.526366
>    Time = 0.4
>    
>    PIMPLE: iteration 1
>    smoothSolver:  Solving for Ux, Initial residual = 0.0112386, Final residual = 2.57986e-06, No Iterations 3
>    smoothSolver:  Solving for Uy, Initial residual = 0.0600746, Final residual = 2.34423e-06, No Iterations 4
>    smoothSolver:  Solving for Uz, Initial residual = 0.0576859, Final residual = 2.06087e-06, No Iterations 4
>    Pressure gradient source: uncorrected Ubar = 0.133673, pressure gradient = -0.000891632
>    GAMG:  Solving for p, Initial residual = 0.201055, Final residual = 0.00409835, No Iterations 2
>    time step continuity errors : sum local = 5.70346e-06, global = -4.59793e-20, cumulative = -4.59793e-20
>    Pressure gradient source: uncorrected Ubar = 0.133669, pressure gradient = -0.000868433
>    GAMG:  Solving for p, Initial residual = 0.0314739, Final residual = 3.67096e-07, No Iterations 8
>    time step continuity errors : sum local = 4.57115e-10, global = -5.07387e-20, cumulative = -9.6718e-20
>    Pressure gradient source: uncorrected Ubar = 0.133668, pressure gradient = -0.000866965
>    smoothSolver:  Solving for k, Initial residual = 0.0654047, Final residual = 1.25654e-06, No Iterations 3
>    bounding k, min: 0 max: 0.000670144 average: 5.74343e-05
>    ExecutionTime = 16.69 s  ClockTime = 40 s
>    .
>    .
>    .
>    ~~~
>    {: .output}
>    - **Important** in an interactive session, the MPI installation that is used is the one present inside the container
>    - This is what we call, use of the internal MPI (MPICH in this case)
>    - For the use of the host MPI installation, the container needs to be executed in "hybrid-mode", as we have explained for the use at Pawsey supercomputers
>    - For executing the singularity container in "hybrid-mode" in your own linux installation, you will need to have a ABI compatible MPICH installed in the host, and follow the instructions from the Singularity documentation (not explained here)
>    - The solver may take several minutes (~10 to ~20 min)
>    - We do not have time to wait, so use `<Ctrl-c>` to kill the solver after a few time steps
>
> 5. Exit the container and check the results in your local disk
>
>    ~~~
>    Singularity> exit
>    ~~~
>    {: .bash}
>
>    ~~~
>    ubuntu@vm:02_*Singularity$ cd run/channel395/
>    ubuntu@vm:channel395$ ls
>    ~~~
>    {: .bash}
>
>    ~~~
>    0  0.org  Allrun  constant  processor0  processor1  processor2  processor3  processor4  system
>    ~~~
>    {: .output}
>    
>    ~~~
>    ubuntu@vm:channel395$ ls -lat processor*/
>    ~~~
>    {: .bash}
>
>    ~~~
>    processor1/:
>    total 32
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:12 0.8
>    drwxr-xr-x  8 ubuntu ubuntu 4096 May 28 12:11 .
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.6
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.4
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:10 0.2
>    drwxr-xr-x  2 ubuntu ubuntu 4096 May 28 11:40 0
>    drwxr-xr-x 11 ubuntu ubuntu 4096 May 28 11:40 ..
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 11:40 constant
>    
>    processor3/:
>    total 32
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:12 0.8
>    drwxr-xr-x  8 ubuntu ubuntu 4096 May 28 12:11 .
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.6
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.4
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:10 0.2
>    drwxr-xr-x  2 ubuntu ubuntu 4096 May 28 11:40 0
>    drwxr-xr-x 11 ubuntu ubuntu 4096 May 28 11:40 ..
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 11:40 constant
>    
>    processor2/:
>    total 32
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:12 0.8
>    drwxr-xr-x  8 ubuntu ubuntu 4096 May 28 12:11 .
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.6
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.4
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:10 0.2
>    drwxr-xr-x  2 ubuntu ubuntu 4096 May 28 11:40 0
>    drwxr-xr-x 11 ubuntu ubuntu 4096 May 28 11:40 ..
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 11:40 constant
>    
>    processor4/:
>    total 32
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:12 0.8
>    drwxr-xr-x  8 ubuntu ubuntu 4096 May 28 12:11 .
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.6
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.4
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:10 0.2
>    drwxr-xr-x  2 ubuntu ubuntu 4096 May 28 11:40 0
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 11:40 constant
>    drwxr-xr-x 11 ubuntu ubuntu 4096 May 28 11:40 ..
>    
>    processor0/:
>    total 32
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:12 0.8
>    drwxr-xr-x  8 ubuntu ubuntu 4096 May 28 12:11 .
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.6
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:11 0.4
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 12:10 0.2
>    drwxr-xr-x  2 ubuntu ubuntu 4096 May 28 11:40 0
>    drwxr-xr-x 11 ubuntu ubuntu 4096 May 28 11:40 ..
>    drwxr-xr-x  3 ubuntu ubuntu 4096 May 28 11:40 constant
>    ~~~
>    {: .output}
>
{: .discussion} 

<p>&nbsp;</p>

## G. Deploy the image to the system of your preference
> ## The image is good to go!
> - Copy it to any system where you want to use it
> - The system will need to have Singularity installed
> - And, in order to run parallel solvers in the most efficient way (hybrid mode), it needs to count with an ABI compatible MPICH
>
{: .callout}

<p>&nbsp;</p>

