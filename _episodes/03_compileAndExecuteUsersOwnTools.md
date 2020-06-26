---
title: "Compile and execute user's own tools"
teaching: 30
exercises: 30
questions:
- How can users execute tools that are not initially installed inside the container?
objectives:
- Compile and execute user's own tools with the compiler and OpenFOAM installation of an existing container
keypoints:
- Define a host directory that will play the role of `WM_PROJECT_USER_DIR`
- For example, `projectUserDir=./anyDirectory`
- Then bind that directory to the path defined inside for `WM_PROJECT_USER_DIR`
- For this exercise, `singularity exec -B $projectUserDir:/home/ofuser/OpenFOAM/ofuser-v1912 $theImage <mySolver> <myOptions>`
---

<p>&nbsp;</p>

## 0. Introduction

> ## Users own tools
>
> - One of the attractive features of OpenFOAM is the possibility of building your own tools/solvers
> - And execute them together with the whole OpenFOAM environment
> - OpenFOAM containers are usually equipped **only** with the standard tools/solvers
> - **Nevertheless, users can still compile their own tools and use them with a standard container**
> - The trick is to **bind** the local host directory to the path where the internal installation looks for user's source files/tools: `WM_PROJECT_USER_DIR`
>
{: .prereq}

<p>&nbsp;</p>

> ## 0.I Accessing the scripts for this episode
>
> In this whole episode, we make use of a series of scripts to cover a typical compilation/execution workflow. Lets start by listing the scripts.
>
> 1. cd into the directory where the provided scripts are. In this case we'll use OpenFOAM-v1912.
> 
>    ~~~
>    zeus-1:~> cd $MYSCRATCH/pawseyTraining/containers-openfoam-workshop-scripts
>    zeus-1:*-scripts> cd 03_compileAndExecuteUsersOwnTools/example_OpenFOAM-v1912
>    zeus-1:*-v1912> ls
>    ~~~
>    {: .bash}
>    
>    ~~~
>    A.cloneMyPimpleFoam.sh    C.extractTutorial.sh  E.decomposeFoam.sh  projectUserDir
>    B.compileMyPimpleFoam.sh  D.adaptCase.sh        F.runFoam.sh        run
>    ~~~
>    {: .output}
>    
> 2. Quickly read one of the scripts, for example `B.compileMyPimpleFoam.sh`.
>    We recommed the following text readers:
>    - `view` (navigate with up and down arrows, use `:q` or `:q!` to quit)
>    - or `less` (navigate with up and down arrows, use `q` to quit) (this one does not have syntax highlight)
>    - If you are using an editor to read the scripts, **DO NOT MODIFY THEM!** because your exercise could mess up
>    - (Try your own settings after succeding with the original exercise, ideally in a copy of the script)
>    
>    ~~~
>    zeus-1:*-v1912> view B.compileMyPimpleFoam.sh
>    ~
>    ~
>    ~
>    :q
>    zeus-1*-v1912>
>    ~~~
>    {: .bash}
>
{: .discussion}

<p>&nbsp;</p>

> ## Sections and scripts for this episode
>
> - In the following sections, there are instructions for submitting these job scripts for execution in the supercomputer one by one:
>    - `A.cloneMyPimpleFoam.sh` **(already pre-executed)** is for copying the source files of an existing solver into our local file system
>    - After the copy, the script above also renames the solver as a user's own solver `myPimpleFoam`
>    - `B.compileMyPimpleFoam.sh` is for compiling user's own solver
>    - `C.extractTutorial.sh` **(already pre-executed)** is for copying a tutorial from the interior of the container into our local file system
>    - `D.adaptCase.sh` **(already pre-executed)** is for modifying the tutorial to comply with Pawsey's best practices 
>    - `E.decomposeFoam.sh` is for meshing and decomposing the intial conditions of the case
>    - `F.runFoam.sh` is for executing the user's own solver
>
{: .prereq}

<p>&nbsp;</p>

> ## So how will this episode flow?
> - The script of section "A" has already been pre-executed.
> - We'll start with a detailed explanation at the end of section "A. Cloning of the solver".
> - And continue the explanation through all section "B. compileMyPimpleFoam".
> - Users will then jump to section E. and proceed by themselves afterwards.
> - At the end, of each section we'll discuss the main instructions within the scripts and the whole process.
>
{: .callout}

<p>&nbsp;</p>

## A. Cloning a standard solver into user's own solver `myPimpleFoam`

> ## The `A.cloneMyPimpleFoam.sh` script
> > ## Main command in the script:   
> >
> > ~~~
> > appDirInside=applications/solvers/incompressible
> > solverOrg=pimpleFoam
> > solverNew=myPimpleFoam
> > srun -n 1 -N 1 singularity exec $theImage bash -c 'cp -r $WM_PROJECT_DIR/'"$appDirInside/$solverOrg $projectUserDir/applications/$solverNew"
> > ~~~
> > {: .language-bash}
> > - The `WM_PROJECT_DIR` variable only exist inside the container, that is why it is being evaluated using the `bash -c` command and the sigle quotes
> > - (See the explanation of the `bash -c` command at the end of this section if needed)
> >
> {: .callout}
> 
> > ## Other important parts of the script:
> > ~~~
> > #3. Define the user directory in the local host and the place where to put the solver
> > #projectUserDir=$MYGROUP/OpenFOAM/$USER-$theVersion/workshop/02_runningUsersOwnTools
> > projectUserDir=$SLURM_SUBMIT_DIR/projectUserDir
> > if ! [ -d $projectUserDir/applications ]; then
> >    mkdir -p $projectUserDir/applications
> > else
> >    echo "The directory $projectUserDir/applications already exists."
> > fi
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #4. Copy the solver from the inside of the container to the local file system
> > appDirInside=applications/solvers/incompressible
> > solverOrg=pimpleFoam
> > solverNew=myPimpleFoam
> > if ! [ -d $projectUserDir/applications/$solverNew ]; then
> >    srun -n 1 -N 1 singularity exec $theImage bash -c 'cp -r $WM_PROJECT_DIR/'"$appDirInside/$solverOrg $projectUserDir/applications/$solverNew"
> > else
> >    echo "The directory $projectUserDir/applications/$solverNew already exists, no new copy has been performed"
> > fi
> > ~~~
> > {: .language-bash}
> > - (See the explanation of the `bash -c` command at the end of this section if needed)
> > 
> > ~~~
> > #5. Going into the new solver directory
> > if [ -d $projectUserDir/applications/$solverNew ]; then
> >    cd $projectUserDir/applications/$solverNew
> >    echo "pwd=$(pwd)"
> > else
> >    echo "For some reason, the directory $projectUserDir/applications/$solverNew, does not exist"
> >    echo "Exiting"; exit 1
> > fi
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #6. Remove not needed stuff
> > echo "Removing not needed stuff"
> > rm -rf *DyMFoam SRFP* *.dep
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #7. Rename the source files and replace words inside for the new solver to be: "myPimpleFoam"
> > echo "Renaming the source files"
> > rename pimpleFoam myPimpleFoam *
> > sed -i 's,pimpleFoam,myPimpleFoam,g' *.C
> > sed -i 's,pimpleFoam,myPimpleFoam,g' *.H
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #8. Modify files inside the Make directory to create the new executable in $FOAM_USER_APPBIN
> > echo "Adapting files inside the Make directory"
> > sed -i 's,pimpleFoam,myPimpleFoam,g' ./Make/files
> > sed -i 's,FOAM_APPBIN,FOAM_USER_APPBIN,g' ./Make/files
> > ~~~
> > {: .language-bash}
> > 
> {: .solution}
{: .solution}

> ## A.I Initial steps for dealing with this section **- [Pre-Executed]**
> 1. Submit the job (no need for reservation as the script uses the `copyq` partition)
> 
>    ~~~
>    zeus-1:*-v1912> sbatch A.cloneMyPimpleFoam.sh 
>    ~~~
>    {: .bash}
> 
>    ~~~
>    Submitted batch job 4632458
>    ~~~
>    {: .output}
{: .solution}

> ## A.II Final steps. To check the source code and important settings for the compilation:
> - Users stuff in the local host is saved under the `./projectUserDir`.
> - At this point, it only has an `applications` folder where the new solver source files are kept: 
>
> 1. Check what is in the `./projectUserDir`
>    ~~~
>    zeus-1:*-v1912> ls projectUserDir/ 
>    ~~~
>    {: .bash}
>   
>    ~~~
>    applications
>    ~~~
>    {: .output}
>   
> 2. cd into the `myPimpleFoam` folder within `applications`
>   
>    ~~~
>    zeus-1:*-v1912> cd projectUserDir/applications/myPimpleFoam
>    zeus-1:myPimpleFoam> ls
>    ~~~
>    {: .bash}
>    
>    ~~~
>    correctPhi.H  createFields.H  Make  myPimpleFoam.C  pEqn.H  UEqn.H
>    ~~~
>    {: .output}
>    - There is our own new solver to be compiled 
>    
> 3. Take a quick look to the source file of the solver:
>   
>    ~~~
>    zeus-1:myPimpleFoam> view myPimpleFoam.C
>    ~
>    ~
>    ~
>    :q
>    ~~~
>    {: .bash}
>    - Discussion of the intrinsics of the solver are out of the scope of this training
>    - But we know it works because it is an exact replica of the standard `pimpleFoam` solver (but renamed)
>    
> 4. Inside the `Make` directory there are two important files:
>   
>    ~~~
>    zeus-1:myPimpleFoam> cd Make
>    zeus-1:Make> ls
>    ~~~
>    {: .bash}
>    
>    ~~~
>    files  options
>    ~~~
>    {: .output}
>   
> 5. In the file `options` there is a list of the OpenFOAM libraries to be included in the solver:
>   
>    ~~~
>    zeus-1:Make> cat options
>    ~~~
>    {: .bash}
>    
>    ~~~
>    EXE_INC = \
>        -I$(LIB_SRC)/finiteVolume/lnInclude \
>        -I$(LIB_SRC)/meshTools/lnInclude \
>        -I$(LIB_SRC)/sampling/lnInclude \
>        -I$(LIB_SRC)/TurbulenceModels/turbulenceModels/lnInclude \
>        -I$(LIB_SRC)/TurbulenceModels/incompressible/lnInclude \
>        -I$(LIB_SRC)/transportModels \
>        -I$(LIB_SRC)/transportModels/incompressible/singlePhaseTransportModel \
>        -I$(LIB_SRC)/dynamicMesh/lnInclude \
>        -I$(LIB_SRC)/dynamicFvMesh/lnInclude
>    
>    EXE_LIBS = \
>        -lfiniteVolume \
>        -lfvOptions \
>        -lmeshTools \
>        -lsampling \
>        -lturbulenceModels \
>        -lincompressibleTurbulenceModels \
>        -lincompressibleTransportModels \
>        -ldynamicMesh \
>        -ldynamicFvMesh \
>        -ltopoChangerFvMesh \
>        -latmosphericModels
>    ~~~
>    {: .output}
>    
> 6. In the file `files` there is a setting for the place where the final executable binary will be created:
>   
>    ~~~
>    zeus-1:Make> cat files
>    ~~~
>    {: .bash}
>    
>    ~~~
>    myPimpleFoam.C
>    
>    EXE = $(FOAM_USER_APPBIN)/myPimpleFoam
>    ~~~
>    {: .output}
{: .discussion}

<p>&nbsp;</p>

> ## A.III The `FOAM_USER_APPBIN`, the `WM_PROJECT_USER_DIR` and other OpenFOAM environment variables
>
> - OpenFOAM makes use of several environmet variables.
> - Most of those variables start with `FOAM_` and `WM_`.
> - The setting of those variables is usually performed by sourcing the `bashrc` file
> - For our images, that `bashrc` file is in `/opt/OpenFOAM/OpenFOAM-v1912/etc`
> - For our Singularity images, this file is sourced every time the container is ran (so no additional sourcing is needed)
> - (If executing the Docker image elsewhere, the you will indeed need to source the `bashrc` file)
>
> <p>&nbsp;</p>
>
> 1. To check a clean list of the variables (defined inside the container) you can use:
>   
>    ~~~
>    zeus-1:myPimpleFoam> module load singularity 
>    zeus-1:myPimpleFoam> theImage=/group/singularity/pawseyRepository/OpenFOAM/openfoam-v1912-pawsey.sif 
>    zeus-1:myPimpleFoam> singularity exec $theImage bash -c 'awk "BEGIN{for(v in ENVIRON) print v}" | egrep "FOAM_|WM_" | sort'
>    ~~~
>    {: .bash}
>    - For anything related to OpenFOAM environmental variables, we recommend the use of the `bash -c` command
>    - (See the explanation of the `bash -c` command at the end of this section if needed)
>   
>    ~~~
>    FOAM_API
>    FOAM_APP
>    FOAM_APPBIN
>    FOAM_ETC
>    FOAM_EXT_LIBBIN
>    FOAM_LIBBIN
>    FOAM_MPI
>    FOAM_RUN
>    FOAM_SETTINGS
>    FOAM_SITE_APPBIN
>    FOAM_SITE_LIBBIN
>    FOAM_SOLVERS
>    FOAM_SRC
>    FOAM_TUTORIALS
>    FOAM_USER_APPBIN
>    FOAM_USER_LIBBIN
>    FOAM_UTILITIES
>    WM_ARCH
>    WM_COMPILER
>    WM_COMPILER_LIB_ARCH
>    WM_COMPILER_TYPE
>    WM_COMPILE_OPTION
>    WM_DIR
>    WM_LABEL_OPTION
>    WM_LABEL_SIZE
>    WM_MPLIB
>    WM_NCOMPPROCS
>    WM_OPTIONS
>    WM_PRECISION_OPTION
>    WM_PROJECT
>    WM_PROJECT_DIR
>    WM_PROJECT_USER_DIR
>    WM_PROJECT_VERSION
>    WM_THIRD_PARTY_DIR
>    ~~~
>    {: .output}
>   
> 2. The main variables of interst here are `WM_PROJECT_USER_DIR`, `FOAM_USER_APPBIN` and `FOAM_USER_LIBBIN`:
>   
>    ~~~
>    zeus-1:myPimpleFoam> singularity exec $theImage bash -c 'printenv | grep "_USER_" | sort -r'
>    ~~~
>    {: .bash}
>   
>    ~~~
>    WM_PROJECT_USER_DIR=/home/ofuser/OpenFOAM/ofuser-v1912
>    FOAM_USER_LIBBIN=/home/ofuser/OpenFOAM/ofuser-v1912/platforms/linux64GccDPInt32Opt/lib
>    FOAM_USER_APPBIN=/home/ofuser/OpenFOAM/ofuser-v1912/platforms/linux64GccDPInt32Opt/bin
>    FOAM_SETTINGS=bash -c printenv | grep "_USER_" | sort -r
>    ~~~
>    {: .output}
>   
>    - `WM_PROJECT_USER_DIR` is the base path where user's stuff is stored 
>    - `FOAM_USER_APPBIN` is the place where the binary executables of user's own solvers are stored and looked for
>    - `FOAM_USER_LIBBIN` is the place where user's own libraries are stored and looked for
>    - As you can see, the APPBIN and LIBBIN paths are under the WM_PROJECT_USER_DIR path
>    - **Internal directories of the container are non-writable**
>    - **But we'll bind a local directory (with `-B` option) to the path of `WM_PROJECT_USER_DIR` to make things work**
{: .discussion}

<p>&nbsp;</p>

> ## The `bash -c` command and the single quotes 
> 
> The `bash` command creates a new bash kernel for execution. Together with the `-c` option, we can define a command to be executed inside that new bash kernel. The use of `bash -c` is very important and useful for the execution of OpenFOAM containers. You will notice that and lear its usage along the several episodes of this workshop.
> 
> For now, we can try to `echo` the content of the `FOAM_TUTORIALS` variable from the command line (non-interactively). First we exemplify some failed attempts and then the use of `bash -c`.
> 
> A first (ineffective) try could be:
> 
> ~~~
> zeus-1:*-v1912> theImage="/group/singularity/pawseyRepository/OpenFOAM/openfoam-v1912-pawsey.sif"
> zeus-1:*-v1912> singularity exec $theImage echo $FOAM_TUTORIALS
> ~~~
> {: .bash}
> 
> ~~~
>
> ~~~
> {: .output}
> - `echo` is ran within the container.
> - But the command did not work because `$FOAM_TUTORIALS` is being evaluated in the host kernel before passing the arguments to the containerxi
> - And, in the host kernel, that variable does not exist:
> 
> ~~~
> zeus-1:*-v1912> echo $FOAM_TUTORIALS
> ~~~
> {: .bash}
> 
> ~~~
>
> ~~~
> {: .output}
>
> <p>&nbsp;</p>
>
> A second (ineffective) try could be:
> 
> ~~~ 
> zeus-1:*-v1912> singularity exec $theImage echo '$FOAM_TUTORIALS'
> ~~~
> {: .bash}
>
> ~~~
> $FOAM_TUTORIALS
> ~~~
> {: .output}
> - Did not work because `echo` now understands to display the exact string but not the content of the variable.
> 
> <p>&nbsp;</p>
>
> A third (ineffective) try could be the use of `bash -c` with double quotes:
> 
> ~~~
> zeus-1:*-v1912> singularity exec $theImage bash -c "echo $FOAM_TUTORIALS"
> ~~~
> {: .bash}
>
> ~~~
>
> ~~~
> {: .output}
> - Almost there, but it did not work because due to the double-apostrophes.
> - Yes, the `echo` command is being evaluated inside the new bash kernel created by the `bash -c` command (inside the container)
> - But the double apostrophes allow the evaluation of the variables inside the quote in the host kernel at the command line.
> - Then, `$FOAM_TUTORIALS` is being evaluated again in the host kernel (where it has no value) before passing the arguments to the container.
> 
> <p>&nbsp;</p>
>
> Finally, this one works (correct use of `bash -c` and single quotes):
> 
> ~~~
> zeus-1:*-v1912> singularity exec $theImage bash -c 'echo $FOAM_TUTORIALS'
> ~~~
> {: .bash}
>
> ~~~
> /opt/OpenFOAM/OpenFOAM-v1912/tutorials
> ~~~
> {: .output}
> - Yes, the `echo` command is being evaluated inside the new bash kernel created by the `bash -c` command (inside the container)
> - The exact string `'echo $FOAM_TUTORIALS'` is passed as an argument without being evaluated by the host kernel.
> - The argument is then received correctly by the new bash kernel inside the container
> - As the variable exists inside the container, then the value can be displayed.
> 
> <p>&nbsp;</p>
>
> Note that without `bash -c` things do not work either:
> 
> ~~~
> zeus-1:*-v1912> singularity exec $theImage 'echo $FOAM_TUTORIALS'
> ~~~
> {: .bash}
>
> ~~~
> /.singularity.d/actions/exec: line 21: exec: echo $FOAM_TUTORIALS: not found
> ~~~
> {: .output}
>
{: .solution}

<p>&nbsp;</p>

## B. Compilation of **myPimpleFoam**

> ## The binding
> - As mentioned above, the trick is to bind a directory in the local host to the internal path where `WM_PROJECT_USER_DIR` point to
> - Inspect the script to check specific command syntax
> 
{: .callout}

> ## The `B.compileMyPimpleFoam.sh` script
> > ## Main command in the script
> >
> > ~~~
> > projectUserDir=$SLURM_SUBMIT_DIR/projectUserDir
> > srun -n 1 -N 1 singularity exec -B $projectUserDir:/home/ofuser/OpenFOAM/ofuser-$theVersion $theImage wmake
> > ~~~
> > {: .language-bash}
> >
> {: .callout}
> 
> > ## Other important parts of the script:
> >
> > ~~~
> > #3. Going into the new solver directory and creating the logs directory
> > projectUserDir=$SLURM_SUBMIT_DIR/projectUserDir
> > solverNew=myPimpleFoam
> > if [ -d $projectUserDir/applications/$solverNew ]; then
> >    cd $projectUserDir/applications/$solverNew
> >    echo "pwd=$(pwd)"
> > else
> >    echo "For some reason, the directory $projectUserDir/applications/$solverNew, does not exist"
> >    echo "Exiting"; exit 1
> > fi
> > logsDir=./logs/compile
> > if ! [ -d $logsDir ]; then
> >    mkdir -p $logsDir
> > fi
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #4. Use container's "wclean" to clean previously existing compilation 
> > echo "Cleaning previous compilation"
> > srun -n 1 -N 1 singularity exec -B $projectUserDir:/home/ofuser/OpenFOAM/ofuser-$theVersion $theImage wclean 2>&1 | tee $logsDir/wclean.$SLURM_JOBID
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #5. Use container's "wmake" (and compiler) to compile your own tool
> > echo "Compiling myPimpleFoam"
> > srun -n 1 -N 1 singularity exec -B $projectUserDir:/home/ofuser/OpenFOAM/ofuser-$theVersion $theImage wmake 2>&1 | tee $logsDir/wmake.$SLURM_JOBID
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #6. Very simple test of the new solver
> > echo "Performing a basic test"
> > singularity exec -B $projectUserDir:/home/ofuser/OpenFOAM/ofuser-$theVersion $theImage myPimpleFoam -help | tee $logsDir/myPimpleFoam.$SLURM_JOBID
> > ~~~
> > {: .language-bash}
> {: .solution}
>
{: .solution}

> ## B.I Steps for dealing with the compilation:
> 1. From the scripts directory, submit the compilation script (use the reservation for the workshop if available):
>    ~~~
>    zeus-1:*-v1912> myReservation=containers 
>    zeus-1:*-v1912> sbatch --reservation=$myReservation B.compileMyPimpleFoam.sh 
>    ~~~
>    {: .bash}
> 
>    > ## If you do not have a reservation
>    > Then, submit normally (or choose the best partition for executing the exercise, the `debugq` for example:)
>    >    ~~~
>    >    zeus-1:*-v1912> sbatch -p debugq B.compileMyPimpleFoam.sh
>    >    ~~~
>    >    {: .bash}
>    >
>    {: .solution}
> 
>    ~~~
>    Submitted batch job 4632558
>    ~~~
>    {: .output}
>
> 2. Check that the new solver binary is now under the `projectUserDir/platforms` tree
>    ~~~
>    zeus-1:*-v1912> ls projectUserDir
>    ~~~
>    {: .bash}
>
>    ~~~
>    applications   platforms
>    ~~~
>    {: .output}
>
>    ~~~
>    zeus-1:*-v1912> ls -la projectUserDir/platforms/linux64GccDPInt32Opt/bin
>    ~~~
>    {: .bash}
>    
>    ~~~
>    total 832
>    drwxrws---+ 2 espinosa pawsey0001   4096 May 24 10:59 .
>    drwxrws---+ 3 espinosa pawsey0001   4096 May 23 20:23 ..
>    -rwxrwx--x+ 1 espinosa pawsey0001 840664 May 24 10:59 myPimpleFoam
>    ~~~
>    {: .output}
>
> 3. You can check the `slurm-4632558.out` or the log file created by `tee` in the job-script (use the SLURM_JOBID of your job):
>
>    ~~~
>    zeus-1:*-v1912> cat projectUserDir/applications/myPimpleFoam/logs/compile/wmake.4632558 
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Making dependency list for source file myPimpleFoam.C
>    g++ -std=c++11 -m64 -DOPENFOAM=1912 -DWM_DP -DWM_LABEL_SIZE=32 -Wall -Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate-depth-100 -I/opt/OpenFOAM/OpenFOAM-v1912/src/finiteVolume/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/meshTools/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/sampling/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/TurbulenceModels/turbulenceModels/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/TurbulenceModels/incompressible/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/transportModels -I/opt/OpenFOAM/OpenFOAM-v1912/src/transportModels/incompressible/singlePhaseTransportModel -I/opt/OpenFOAM/OpenFOAM-v1912/src/dynamicMesh/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/dynamicFvMesh/lnInclude -IlnInclude -I. -I/opt/OpenFOAM/OpenFOAM-v1912/src/OpenFOAM/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/OSspecific/POSIX/lnInclude   -fPIC -c myPimpleFoam.C -o Make/linux64GccDPInt32Opt/myPimpleFoam.o
>    g++ -std=c++11 -m64 -DOPENFOAM=1912 -DWM_DP -DWM_LABEL_SIZE=32 -Wall -Wextra -Wold-style-cast -Wnon-virtual-dtor -Wno-unused-parameter -Wno-invalid-offsetof -Wno-attributes -Wno-unknown-pragmas -O3  -DNoRepository -ftemplate-depth-100 -I/opt/OpenFOAM/OpenFOAM-v1912/src/finiteVolume/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/meshTools/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/sampling/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/TurbulenceModels/turbulenceModels/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/TurbulenceModels/incompressible/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/transportModels -I/opt/OpenFOAM/OpenFOAM-v1912/src/transportModels/incompressible/singlePhaseTransportModel -I/opt/OpenFOAM/OpenFOAM-v1912/src/dynamicMesh/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/dynamicFvMesh/lnInclude -IlnInclude -I. -I/opt/OpenFOAM/OpenFOAM-v1912/src/OpenFOAM/lnInclude -I/opt/OpenFOAM/OpenFOAM-v1912/src/OSspecific/POSIX/lnInclude   -fPIC -Xlinker --add-needed -Xlinker --no-as-needed Make/linux64GccDPInt32Opt/myPimpleFoam.o -L/opt/OpenFOAM/OpenFOAM-v1912/platforms/linux64GccDPInt32Opt/lib \
>        -lfiniteVolume -lfvOptions -lmeshTools -lsampling -lturbulenceModels -lincompressibleTurbulenceModels -lincompressibleTransportModels -ldynamicMesh -ldynamicFvMesh -ltopoChangerFvMesh -latmosphericModels -lOpenFOAM -ldl  \
>         -lm -o /home/ofuser/OpenFOAM/ofuser-v1912/platforms/linux64GccDPInt32Opt/bin/myPimpleFoam
>    ~~~
>    {: .output}
>
> 4. You can check the log for the mini-test of the solver:
>
>    ~~~
>    zeus-1:*-v1912> cat projectUserDir/applications/myPimpleFoam/logs/compile/myPimpleFoam.4632558 
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Usage: myPimpleFoam [OPTIONS]
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
>    Transient solver for incompressible, turbulent flow of Newtonian fluids on a
>    moving mesh.
>    
>    Using: OpenFOAM-v1912 (1912) - visit www.openfoam.com
>    Build: _f3950763fe-20191219
>    Arch:  LSB;label=32;scalar=64
>    ~~~
>    {: .output}
>
> 5. Or you can perform that basic mini-test from the command line (binding the local `projectUserDir` directory):
>
>    ~~~
>    zeus-1:*-v1912> module load singularity
>    zeus-1:*-v1912> theImage=/group/singularity/pawseyRepository/OpenFOAM/openfoam-v1912-pawsey.sif
>    zeus-1:*-v1912> singularity exec -B ./projectUserDir:/home/ofuser/OpenFOAM/ofuser-v1912 $theImage myPimpleFoam -help 
>    ~~~
>    {: .bash}
>
>    ~~~
>    Usage: myPimpleFoam [OPTIONS]
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
>    Transient solver for incompressible, turbulent flow of Newtonian fluids on a
>    moving mesh.
>    
>    Using: OpenFOAM-v1912 (1912) - visit www.openfoam.com
>    Build: _f3950763fe-20191219
>    Arch:  LSB;label=32;scalar=64
>    ~~~
>    {: .output} 
>
> 6. (If you do not use the binding, the solver will not be found):
>
>    ~~~
>    zeus-1:*-v1912> singularity exec $theImage myPimpleFoam -help 
>    ~~~
>    {: .bash}
>
>    ~~~
>    /.singularity.d/actions/exec: line 21: exec: myPimpleFoam: not found
>    ~~~
>    {: .output} 
{: .discussion}

<p>&nbsp;</p>

## C. Extraction of the tutorial: channel395
   
> ## The `C.extractTutorial.sh` script (main parts to be discussed):
>
> ~~~
> #!/bin/bash -l
> #SBATCH --export=NONE
> #SBATCH --time=00:05:00
> #SBATCH --ntasks=1
> #SBATCH --partition=copyq #Ideally, you should be using the copyq for this kind of processes
> ~~~
> {: .language-bash}
> 
> ~~~
> #4. Copy the tutorialCase to the workingDir
> if ! [ -d $caseDir ]; then
>    srun -n 1 -N 1 singularity exec $theImage bash -c 'cp -r $FOAM_TUTORIALS/'"$tutorialCase $caseDir"
> else
>    echo "The case=$caseDir already exists, no new copy has been performed"
> fi
> ~~~
> {: .language-bash}
> 
{: .solution}

> ## C.I Steps for dealing with the extraction of the "channel395" case: **- [Pre-Executed]**
> 1. Submit the job (no need for reservation as the script uses the `copyq` partition)
> 
>    ~~~
>    zeus-1:*-v1912> sbatch C.extractTutorial.sh 
>    ~~~
>    {: .bash}
> 
>    ~~~
>    Submitted batch job 4632458
>    ~~~
>    {: .output}
> 
> 2. Check its status in the queue:
> 
>    ~~~
>    zeus-1:*-v1912> squeue -u $USER
>    ~~~
>    {: .bash}
> 
>    ~~~
>    JOBID    USER     ACCOUNT     PARTITION            NAME EXEC_HOST ST     REASON   START_TIME     END_TIME  TIME_LEFT NODES   PRIORITY
>    4632468  espinosa pawsey0001  workq      C.extractTutor       n/a PD       None          N/A          N/A       5:00     1      75190
>    ~~~
>    {: .output}
>
> - If no status is shown, it may have finished execution already.
> 
> 3. Check that the tutorial has been copied to our host file system
> 
>    ~~~
>    zeus-1:*-v1912> ls ./run/channel395/
>    ~~~
>    {: .bash}
> 
>    ~~~
>    0  0.orig  Allrun  constant  system
>    ~~~
>    {: .output}
{: .solution}

<p>&nbsp;</p>

## D. Adapt the case to your needs (and Pawsey best practices)

> ## The `D.adaptCase.sh` script (main parts to be discussed):
>
> ~~~
> #5. Defining OpenFOAM controlDict settings for Pawsey Best Practices
> ##5.1 Replacing writeFormat, runTimeModifiable and purgeRight settings
> foam_writeFormat="binary"
> sed -i 's,^writeFormat.*,writeFormat    '"$foam_writeFormat"';,' ./system/controlDict
> foam_runTimeModifiable="false"
> sed -i 's,^runTimeModifiable.*,runTimeModifiable    '"$foam_runTimeModifiable"';,' ./system/controlDict
> foam_purgeWrite=10
> sed -i 's,^purgeWrite.*,purgeWrite    '"$foam_purgeWrite"';,' ./system/controlDict
> ~~~
> {: .language-bash}
> 
> ~~~
> ##5.2 Defining the use of collated fileHandler of output results 
> echo "OptimisationSwitches" >> ./system/controlDict
> echo "{" >> ./system/controlDict
> echo "   fileHandler collated;" >> ./system/controlDict
> echo "}" >> ./system/controlDict
> ~~~
> {: .language-bash}
{: .solution}

> ## D.I Steps for dealing with the adaptation of the case: **- [Pre-Executed]**
>
> 1. Submit the adaptation script
> 
>    ~~~
>    zeus-1:*-v1912> sbatch D.adaptCase.sh 
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Submitted batch job 4632548
>    ~~~
>    {: .output}
> 2. Check the adapted settings
>    ~~~ 
>    zeus-1:*-v1912> cd run/channel395
>    zeus-1:channel395> ls
>    ~~~
>    {: .bash}
> 
>    ~~~
>    0  0.orig  Allrun  constant  system
>    ~~~
>    {: .output}
>
>    - Initial conditions are in the subdirectory `0` 
>    - Mesh and fluid prperties definition are under the tree of the subdirectory `constant`
>    - Solver settings are in the "dictionaries" inside the `system` subdirectory 
>
>    Read the `controlDict` dictionary:
> 
>    ~~~
>    zeus-1:channel395> view system/controlDict
>    ~
>    ~
>    ~
>    :q
>    ~~~
>    {: .bash}
> 
>    > ## The settings that were adapted in `..../run/channel395/system/controlDict` 
>    > 
>    > - To keep only a few result directories at a time (10 maximum in this case)
>    >    ~~~
>    >    purgeWrite      10;
>    >    ~~~
>    > 
>    > - To use binary writing format to accelerate writing and reduce the size of the files
>    > 
>    >    ~~~
>    >    writeFormat     binary;
>    >    ~~~
>    > 
>    > - Never use `runTimeModifiable`. This option creates permanent reading of dictionaries (each time step) which overloads the shared file system.
>    > 
>    >    ~~~
>    >    runTimeModifiable false;
>    >    ~~~
>    > 
>    > - If version is higher-or-equal than OpenFOAM-6 or OpenFOAM-v1812, always use the collated option
>    > 
>    >    ~~~
>    >    optimisationSwitches
>    >    {
>    >        fileHandler collated;
>    >    }
>    >    ~~~
>    {: .solution}
{: .solution} 

<p>&nbsp;</p>

## E. Decomposition
> ## The `E.decomposeFoam.sh` script (main parts to be discussed):
> ~~~
> #!/bin/bash -l
> #SBATCH --ntasks=1
> #SBATCH --mem=4G
> #SBATCH --ntasks-per-node=28
> #SBATCH --clusters=zeus
> #SBATCH --partition=workq
> #SBATCH --time=0:10:00
> #SBATCH --export=none
> ~~~
> {: .language-bash}
>
> ~~~
> #6. Defining the ioRanks for collating I/O
> # groups of 2 for this exercise (please read our documentation for the recommendations for production runs)
> export FOAM_IORANKS='(0 2 4 6)'
> ~~~
> {: .language-bash}
> 
> ~~~
> #7. Perform all preprocessing OpenFOAM steps up to decomposition
> echo "Executing blockMesh"
> srun -n 1 -N 1 singularity exec $theImage blockMesh 2>&1 | tee $logsDir/log.blockMesh.$SLURM_JOBID
> echo "Executing decomposePar"
> srun -n 1 -N 1 singularity exec $theImage decomposePar -cellDist -force 2>&1 | tee $logsDir/log.decomposePar.$SLURM_JOBID
> ~~~
> {: .language-bash}
{: .solution}

> ## E.I Steps for dealing with decomposition:
> 
> 1. Submit the decomposition script from the scripts directory (use the reservation for the workshop if available)
>
>    ~~~
>    zeus-1:*-v1912> myReservation=containers
>    zeus-1:*-v1912> sbatch --reservation=$myReservation E.decomposeFoam.sh 
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Submitted batch job 4632558
>    ~~~
>    {: .output}
> 
> 2. Check that the decomposition has been performed:
> 
>    ~~~
>    zeus-1:*-v1912> ls ./run/channel395/processor*
>    ~~~
>    {: .bash}
> 
>    ~~~
>    ./run/channel395/processors4_0-1:
>    0  constant
>    
>    ./run/channel395/processors4_2-3:
>    0  constant
>    ~~~
>    {: .output}
>
> 3. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/pre/`
{: .challenge}

<p>&nbsp;</p>

## F. Executing the **NEW** solver **"myPimpleFoam"**

> ## The binding
> - Again, the important concept here is the binding of the local folder for the container to be able to read the binary executable
> - And, obviously, to call the own solver: `myPimpleFoam` to perform the solution
> - Check the interior of the scripts for especific command syntax
>
{: .callout}

> ## The `F.runFoam.sh` script 
> > ## Main command in the script:
> > ~~~
> > of_solver=myPimpleFoam
> > projectUserDir=$SLURM_SUBMIT_DIR/projectUserDir
> > srun -n $SLURM_NTASKS -N $SLURM_JOB_NUM_NODES singularity exec -B $projectUserDir:/home/ofuser/OpenFOAM/ofuser-$theVersion $theImage $of_solver -parallel 
> > ~~~
> > {: .language-bash}
> {: .callout}
> 
> > ## Other important parts of the script:
> > ~~~
> > #SBATCH --ntasks=4
> > #SBATCH --mem=16G
> > #SBATCH --ntasks-per-node=28
> > #SBATCH --cluster=zeus
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #5. Reading OpenFOAM decomposeParDict settings
> > foam_numberOfSubdomains=$(grep "^numberOfSubdomains" ./system/decomposeParDict | tr -dc '0-9')
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #7. Checking if the number of tasks coincide with the number of subdomains
> > if [[ $foam_numberOfSubdomains -ne $SLURM_NTASKS ]]; then
> >    echo "foam_numberOfSubdomains read from ./system/decomposeParDict is $foam_numberOfSubdomains"
> >    echo "and"
> >    echo "SLURM_NTASKS in this job is $SLURM_NTASKS"
> >    echo "These should be the same"
> >    echo "Therefore, exiting this job"
> >    echo "Exiting"; exit 1
> > fi
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #8. Defining OpenFOAM controlDict settings for this run
> > foam_startFrom=startTime
> > #foam_startFrom=latestTime
> > foam_startTime=0
> > #foam_startTime=15
> > foam_endTime=10
> > #foam_endTime=30
> > foam_writeInterval=1
> > foam_purgeWrite=10
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #9. Changing OpenFOAM controlDict settings
> > sed -i 's,^startFrom.*,startFrom    '"$foam_startFrom"';,' system/controlDict
> > sed -i 's,^startTime.*,startTime    '"$foam_startTime"';,' system/controlDict
> > sed -i 's,^endTime.*,endTime    '"$foam_endTime"';,' system/controlDict
> > sed -i 's,^writeInterval.*,writeInterval    '"$foam_writeInterval"';,' system/controlDict
> > sed -i 's,^purgeWrite.*,purgeWrite    '"$foam_purgeWrite"';,' system/controlDict
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #10. Defining the solver
> > of_solver=myPimpleFoam
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #11. Defining the projectUserDir to be mounted into the path of the internal WM_PROJECT_USER_DIR
> > projectUserDir=$SLURM_SUBMIT_DIR/projectUserDir
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #12. Execute the case 
> > echo "About to execute the case"
> > srun -n $SLURM_NTASKS -N $SLURM_JOB_NUM_NODES singularity exec -B $projectUserDir:/home/ofuser/OpenFOAM/ofuser-$theVersion $theImage $of_solver -parallel 2>&1 | tee $logsDir/log.$theSolver.$SLURM_JOBID
> > echo "Execution finished"
> > ~~~
> > {: .language-bash}
> >
> {: .solution}
>
{: .solution}

> ## F.I Steps for dealing with the solver
> 1. Submit the solver script (from the scripts directory)
> 
>    ~~~
>    zeus-1:*-v1912> sbatch --reservation=$myReservation F.runFoam.sh 
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Submitted batch job 4632685 on cluster zeus
>    ~~~
>    {: .output}
> 
> 2. Check that the solver is running:
> 
>    ~~~
>    zeus-1:*-v1912> squeue -u $USER
>    ~~~
>    {: .bash}
> 
>    ~~~
>    JOBID    USER     ACCOUNT     PARTITION            NAME EXEC_HOST ST     REASON   START_TIME     END_TIME  TIME_LEFT NODES   PRIORITY
>    4632685  espinosa pawsey0001  workq        F.runFoam.sh       n/a PD  Resources     17:09:28     17:19:28      10:00     1      75190 
>    ~~~
>    {: .output}
> 
> 3. Observe the output of the job with `tail -f` at runtime (use `<Ctrl-C>` to exit the command):
>    ~~~
>    zeus-1:*-v1912> tail -f slurm-4632685.out
>    ~~~
>    {: .bash}
> 
>    ~~~
>    .
>    .
>    .
>    Time = 0.2
>    
>    PIMPLE: iteration 1
>    smoothSolver:  Solving for Ux, Initial residual = 0.0118746, Final residual = 1.89249e-06, No Iterations 3
>    smoothSolver:  Solving for Uy, Initial residual = 0.0617212, Final residual = 1.68113e-06, No Iterations 4
>    smoothSolver:  Solving for Uz, Initial residual = 0.0589944, Final residual = 9.70923e-06, No Iterations 3
>    Pressure gradient source: uncorrected Ubar = 0.13369, pressure gradient = -0.000964871
>    GAMG:  Solving for p, Initial residual = 0.213844, Final residual = 0.00414884, No Iterations 2
>    time step continuity errors : sum local = 5.82807e-06, global = -1.41211e-19, cumulative = -1.41211e-19
>    Pressure gradient source: uncorrected Ubar = 0.133687, pressure gradient = -0.000947989
>    GAMG:  Solving for p, Initial residual = 0.0222643, Final residual = 4.30412e-07, No Iterations 7
>    time step continuity errors : sum local = 5.63638e-10, global = -2.40486e-19, cumulative = -3.81697e-19
>    Pressure gradient source: uncorrected Ubar = 0.133687, pressure gradient = -0.000947874
>    ExecutionTime = 0.25 s  ClockTime = 0 s
>    .
>    .
>    .
>    ~~~
>    {: .output}
>    - Press `<Ctrl-C>` to exit `tail`
> 
> 3. Check that the solver gave some results:
> 
>    ~~~
>    zeus-1:*-v1912> ls ./run/channel395/processor*
>    ~~~
>    {: .bash}
> 
>    ~~~
>    ./run/channel395/processors4_0-1:
>    0  10  8.2  8.4  8.6  8.8  9  9.2  9.4  9.6  9.8  constant
>    
>    ./run/channel395/processors4_2-3:
>    0  10  8.2  8.4  8.6  8.8  9  9.2  9.4  9.6  9.8  constant
>    ~~~
>    {: .output}
>
> 4. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/run/`
{: .challenge}

<p>&nbsp;</p>

## Z. Further notes on how to use OpenFOAM and OpenFOAM containers at Pawsey

The usage of OpenFOAM and OpenFOAM containers at Pawsey has already been described in our documentation: [OpenFOAM documentation at Pawsey](https://support.pawsey.org.au/documentation/display/US/OpenFOAM)

and in a technical newsletter note: [https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04](https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04) 

<p>&nbsp;</p>
