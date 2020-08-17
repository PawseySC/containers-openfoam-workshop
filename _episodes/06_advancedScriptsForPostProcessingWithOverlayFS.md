---
title: "Advanced scripts for postprocessing with OverlayFS"
teaching: 30
exercises: 30
questions:
- How can I postprocess a large number of results?
objectives:
- Use provided scripts to postprocess a large number of results without increasing dramatically the existing number of files in the system
keypoints:
- No, unfortunately a container cannot mount more than 1 OverlayFS file at the same time
- Yes, this implies that the results need to be copied back to the host file system before reconstruction
- In order to avoid the presence of many files in the host, this should be done by small batches
    - 1.Copy small batch of results from the interior of the `./overlayFSDir/overlay*` files towards the `./bakDir/bak.processor*` directories in the host file system
    - 2.Now create `processor*` soft links to point to `./bakDir/bak.processor*` directories and not to the directory structure inside the OverlayFS files
    - 3.Reconstruct that small batch
    - 4.Remove the decomposed result-times from the `./bakDir/bak.processor*` directories. Only the fully reconstructed result-times are kept in the host. And the original decomposed results are only kept inside the OverlayFS files.
    - 5.Continue the cycle until postprocessing all the result-times needed
---

<p>&nbsp;</p>

## 0. Introduction

> ## Postprocessing results in batches
> - The workflow presented here is extremelly similar to that in the previous episode (please study previous one first)
> - Some minor differences in this episode are: 1) the `overlay*` files are kept in the directory `./overlayFSDir`, and 2) the renamed `bak.processor*` directories are moved to the directory `./bakDir`
> - And **the main difference** from the previous episode is: the reconstruction of the existing results can handle many time-results in the same job
> - Nevertheless, in order to avoid the creation of a large number of files in the host file system, the extraction of the result-times from the overlayFS files is performed in batches of small size
> - All the decomposed result-times of each small batch are postprocessed (reconstructed) and then deleted before the following batch is extracted/processed
>
{: .prereq}

> ## Use of bash functions
> - Another difference in the workflow scripts for this episode is the use of bash functions
> - These functions are defined in a separate bash script contained in the directory named `../../A1_auxiliaryScripts`
> - That script is sourced from all the workflow (A,B,C..G) scripts
> - In practice, users should keep this directory in a known location and source the definition of bash functions from there
>
{: .callout}

> ## 0.I Accessing the scripts for this episode
>
> In this episode, we make use of a series of scripts to cover a typical compilation/execution workflow. Lets start by listing the scripts.
>
>
> 1. List the content of the `A1_auxiliaryScripts` directory:
>
>    ~~~
>    zeus-1:~> cd $MYSCRATCH/pawseyTraining/containers-openfoam-workshop-scripts
>    zeus-1:*-scripts> ls A1_auxiliaryScripts
>    ~~~
>    {: .bash}
>    
>    ~~~
>    ofContainersOverlayFunctions.sh
>    ~~~
>    {: .output}
>    - This file contains the definition of several functions utilised within the workflow scripts
>    
> <p>&nbsp;</p>
>
> 2. cd into the directory that contains the scripts for executing the workflow of this exercise. In this case we'll use OpenFOAM-2.4.x.
> 
>    ~~~
>    zeus-1:*-scripts> cd 06_advancedScriptsForPostProcessingWithOverlayFS/example_OpenFOAM-2.4.x
>    zeus-1:*-2.4.x> ls
>    ~~~
>    {: .bash}
>    
>    ~~~
>    A.extractAndAdpatTutorial.sh  C.setupOverlayFoam.sh        F.extractFromOverlayIntoBak.sh  run
>    B.decomposeFoam.sh            D.runFoam.sh                 G.reconstructFromBak.sh
>    caseSettingsFoam.sh           E.reconstructFromOverlay.sh  imageSettingsSingularity.sh
>    ~~~
>    {: .output}
>
{: .discussion}

<p>&nbsp;</p>

> ## Sections and scripts for this episode
>
> - In the following sections, there are instructions for submitting these workflow scripts for execution in the supercomputer one by one:
>    - `A.extractAndAdaptTutorial.sh` **(already pre-executed)** is for copying an adapting a case to solve
>    - `B.decomposeFoam.sh` is decomposing the case to solve
>    - `C.setupOverlayFoam.sh` is for creating the OverlayFS files to store the result
>    - `D.runFoam.sh` is for executing the solver (and writing results to the interior of the overlay files) 
>    - `E.reconstructFromOverlay.sh` is for reconstructing several result-times in batches
>    - `F.extractFromOverlayIntoBak.sh` is for extracting a batch of result-times from the overlay files (no reconstruction)
>    - `G.reconstructFromBak.sh` is for reconstructing existing results in the `bak.processor*` directories
>
>    <p>&nbsp;</p>
>
>    - `caseSettingsFoam.sh` is a script that defines the settings for using OpenFOAM within all the workflow scripts (it is being sourced from all the A,B,C,..G scripts)
>    - `imageSettingsSingularity.sh` is a script that defines the settings for using Singularity within all the other scripts (it is being sourced from all the workflow scripts)
>
>    <p>&nbsp;</p>
>
>    - `../../A1_auxiliaryScripts/ofContainersOverlayFunctions.sh` is the script that contains the definition of the bash functions utilised in the workflow scripts (This script is sourced from all the workflow scripts)
>
{: .prereq}

<p>&nbsp;</p>

> ## So how will this episode flow?
> - The script of section "A. has already been pre-executed.
> - The usage of these scripts is extremely similar to that of previous episode (we recommend to practise previous episode first)
> - Additional explanations are dedicated to the use of the bash functions
> - And on the reconstruction made in batches (section E.)
> - Sections F. and G. are left to the user to explore by themselves
>
{: .callout}

<p>&nbsp;</p>

## A. Extract and adapt the tutorial to be solved **- [Pre-Executed]**
   
> ## The `A.extractAndAdaptTutorial.sh` script (main parts to be discussed):
>
> ~~~
> #1. Loading the container settings, case settings and auxiliary functions (order is important)
> source $SLURM_SUBMIT_DIR/imageSettingsSingularity.sh
> source $SLURM_SUBMIT_DIR/caseSettingsFoam.sh
> overlayFunctionsScript=$auxScriptsDir/ofContainersOverlayFunctions.sh
> ~~~
> {: .language-bash}
>
> > ## The `imageSettingsSingularity.sh` script (main sections to be discussed):
> >
> > ~~~
> > #Module environment
> > module load singularity
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #Defining the container to be used
> > theRepo=/group/singularity/pawseyRepository/OpenFOAM
> > theContainerBaseName=openfoam
> > theVersion=2.4.x
> > theProvider=pawsey
> > theImage=$theRepo/$theContainerBaseName-$theVersion-$theProvider.sif
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #Defining the path of the auxiliary scripts for dealing with overlayFS
> > #(Define the path to a more permanent directory for production workflows)
> > auxScriptsDir=$SLURM_SUBMIT_DIR/../../A1_auxiliaryScripts
> > ~~~
> > {: .language-bash}
> >
> {: .solution}
>
> > ## The `caseSettingsFoam.sh` script (main sections to be discussed):
> > 
> > ~~~
> > #Choosing the tutorial case
> > tutorialAppDir=incompressible/pimpleFoam
> > tutorialName=channel395
> > tutorialCase=$tutorialAppDir/$tutorialName
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #Choosing the working directory for the case to solve
> > baseWorkingDir=$SLURM_SUBMIT_DIR/run
> > if ! [ -d $baseWorkingDir ]; then
> >     echo "Creating baseWorkingDir=$baseWorkingDir"
> >     mkdir -p $baseWorkingDir
> > fi
> > caseName=$tutorialName
> > caseDir=$baseWorkingDir/$caseName
> > ~~~
> > {: .language-bash}
> {: .solution}
{: .solution}

> ## A.I Steps for dealing with the extraction and adaptation of the case to be solved
> 1. Submit the job (no need for reservation as the script uses the `copyq` partition)
> 
>    ~~~
>    zeus-1:*-2.4.x> sbatch A.extractAndAdaptTutorial.sh 
>    ~~~
>    {: .bash}
> 
>    ~~~
>    Submitted batch job 4632758
>    ~~~
>    {: .output}
> 
> 2. Check that the tutorial has been copied to our host file system
> 
>    ~~~
>    zeus-1:*-2.4.x> ls ./run/channel395/
>    ~~~
>    {: .bash}
> 
>    ~~~
>    0  0.orig  Allrun  constant  system
>    ~~~
>    {: .output}
>
> 3. Read the `controlDict` dictionary:
> 
>    ~~~
>    zeus-1:*-2.4.x> view ./run/channel395/system/controlDict
>    ~
>    ~
>    ~
>    :q
>    ~~~
>    {: .bash}
> 
>    > ## The settings that were adapted in `./run/channel395/system/controlDict` 
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
>    {: .solution}
{: .solution} 

<p>&nbsp;</p>

## B. Decomposition
> ## The `B.decomposeFoam.sh` script (main points to be discussed):
> ~~~
> #1. Loading the container settings, case settings and auxiliary functions (order is important)
> source $SLURM_SUBMIT_DIR/imageSettingsSingularity.sh
> source $SLURM_SUBMIT_DIR/caseSettingsFoam.sh
> overlayFunctionsScript=$auxScriptsDir/ofContainersOverlayFunctions.sh
> ~~~
> {: .language-bash}
>
> > ## The `imageSettingsSingularity.sh` script (main sections to be discussed):
> >
> > ~~~
> > #Module environment
> > module load singularity
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #Defining the container to be used
> > theRepo=/group/singularity/pawseyRepository/OpenFOAM
> > theContainerBaseName=openfoam
> > theVersion=2.4.x
> > theProvider=pawsey
> > theImage=$theRepo/$theContainerBaseName-$theVersion-$theProvider.sif
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #Defining the path of the auxiliary scripts for dealing with overlayFS
> > #(Define the path to a more permanent directory for production workflows)
> > auxScriptsDir=$SLURM_SUBMIT_DIR/../../A1_auxiliaryScripts
> > ~~~
> > {: .language-bash}
> >
> {: .solution}
>
> > ## The `caseSettingsFoam.sh` script (main sections to be discussed):
> > ~~~
> > #Choosing the working directory for the case to solve
> > baseWorkingDir=$SLURM_SUBMIT_DIR/run
> > if ! [ -d $baseWorkingDir ]; then
> >     echo "Creating baseWorkingDir=$baseWorkingDir"
> >     mkdir -p $baseWorkingDir
> > fi
> > caseName=$tutorialName
> > caseDir=$baseWorkingDir/$caseName
> > ~~~
> > {: .language-bash}
> {: .solution} 
> 
> ~~~
> #7. Perform all preprocessing OpenFOAM steps up to decomposition
> srun -n 1 -N 1 singularity exec $theImage blockMesh 2>&1 | tee $logsDir/log.blockMesh
> srun -n 1 -N 1 singularity exec $theImage decomposePar -cellDist 2>&1 | tee $logsDir/log.decomposePar
> ~~~
> {: .bash}
{: .solution}

> ## B.I Steps for dealing with decomposition:
> 
> 1. Submit the decomposition script from the scripts directory (use the reservation for the workshop if available)
>
>    ~~~
>    zeus-1:*-2.4.x> myReservation=containers
>    ~~~
>    {: .bash}
> 
>    ~~~
>    zeus-1:*-2.4.x> sbatch --reservation=$myReservation B.decomposeFoam.sh 
>    ~~~
>    {: .bash}
>    - If a reservation is not available, do not use the option. (Or you can use the debugq: `--partition=debugq` instead.)
>    
>    ~~~
>    Submitted batch job 4632558
>    ~~~
>    {: .output}
> 
> 2. Check that the decomposition has been performed:
> 
>    ~~~
>    zeus-1:*-2.4.x> ls ./run/channel395/processor*
>    ~~~
>    {: .bash}
> 
>    ~~~
>    ./run/channel395/processor0:
>    0  constant
>    
>    ./run/channel395/processor1:
>    0  constant
>    
>    ./run/channel395/processor2:
>    0  constant
>    
>    ./run/channel395/processor3:
>    0  constant
>    
>    ./run/channel395/processor4:
>    0  constant
>    ~~~
>    {: .output}
>    - Note that one `processor*` directory was created per `numberOfSubdomains` (The number of subdomains is set in the `system/decomposeParDict` dictionary)
>    - Also note that this tutorial uses 5 subdomains (and 5 cores when executing the solver (below))
>
> 3. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/pre/`
{: .discussion}

<p>&nbsp;</p>

## C. Setup the overlayFS
> ## The `C.setupOverlayFoam.sh` script (main points to be discussed):
>
> ~~~
> #SBATCH --ntasks=4 #Several tasks will be used for copying files. (Independent from the numberOfSubdomains)
> ~~~
> {: .language-bash}
>
> ~~~
> #1. Loading the container settings, case settings and auxiliary functions (order is important)
> source $SLURM_SUBMIT_DIR/imageSettingsSingularity.sh
> source $SLURM_SUBMIT_DIR/caseSettingsFoam.sh
> overlayFunctionsScript=$auxScriptsDir/ofContainersOverlayFunctions.sh
> ~~~
> {: .language-bash}
>
> > ## The `imageSettingsSingularity.sh` script (main sections to be discussed):
> > ~~~
> > #Module environment
> > module load singularity
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #Defining the container to be used
> > theRepo=/group/singularity/pawseyRepository/OpenFOAM
> > theContainerBaseName=openfoam
> > theVersion=2.4.x
> > theProvider=pawsey
> > theImage=$theRepo/$theContainerBaseName-$theVersion-$theProvider.sif
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #Defining settings for the OverlayFS
> > overlaySizeGb=1
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #Defining the path of the auxiliary scripts for dealing with overlayFS
> > #(Define the path to a more permanent directory for production workflows)
> > auxScriptsDir=$SLURM_SUBMIT_DIR/../../A1_auxiliaryScripts
> > ~~~
> > {: .language-bash}
> >
> {: .solution}
>
> > ## The `caseSettingsFoam.sh` script (main sections to be discussed):
> > ~~~
> > #Choosing the working directory for the case to solve
> > baseWorkingDir=$SLURM_SUBMIT_DIR/run
> > if ! [ -d $baseWorkingDir ]; then
> >     echo "Creating baseWorkingDir=$baseWorkingDir"
> >     mkdir -p $baseWorkingDir
> > fi
> > caseName=$tutorialName
> > caseDir=$baseWorkingDir/$caseName
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #Defining the name of the directory inside the overlay* files at which results will be saved
> > baseInsideDir=/overlayOpenFOAM/run
> > insideName=$caseName
> > insideDir=$baseInsideDir/$insideName
> > ~~~
> > {: .language-bash}
> {: .solution}
>
> ~~~
> #3. Create the directory where the OverlayFS files are going to be kept
> if ! [ -d ./overlayFSDir ]; then
>    echo "Creating the directory ./overlayFSDir which will contain the overlay* files:"
>    mkdir -p ./overlayFSDir
> else
>    echo "For some reason, the directory ./overlayFSDir for saving the overlay* files already exists:"
>    echo "Warning:No creation needed"
> fi
> ~~~
> {: .language-bash}
> 
> ~~~
> #5. Rename the processor* directories into bak.processor* and move them into ./bakDir
> #(OpenFOAM wont be able to see these directories)
> #(Access will be performed through soft links)
> echo "Renaming the processor directories"
> rename processor bak.processor processor*
> if ! [ -d ./bakDir ]; then
>    echo "Creating the directory ./bakDir that will contain the bak.processor* directories:"
>    mkdir -p ./bakDir
> else
>    echo "For some reason, the directory ./bakDir for containing the bak.processor* dirs already exists:"
>    echo "Warning:No creation needed"
> fi
> 
> if ! [ -d ./bakDir/bak.processor0 ]; then
>    echo "Moving all bak.processor* directories into ./bakDir"
>    mv bak.processor* ./bakDir
> else
>    echo "The directory ./bakDir/bak.processor0 already exists"
>    echo "No move/replacement of bak.processor* directories will be performed"
>    echo "Exiting";exit 1
> fi
> ~~~
> {: .language-bash}
> 
> ~~~
> #6. Creating a first ./overlayFSDir/overlayII file (./overlayFSDir/overlay0)
> createOverlay0 $overlaySizeGb;success=$? #Calling the function for creating the ./overlayFSDir/overlay0 file
> if [ $success -eq 222 ]; then 
>    echo "./overlayFSDir/overlay0 already exists"
>    echo "Exiting";exit 1
> elif [ $success -ne 0 ]; then 
>    echo "Failed creating ./overlayFSDir/overlay0, exiting"
>    echo "Exiting";exit 1
> fi
> ~~~
> {: .language-bash}
> - Note the use of the `createOverlay0` function
> - The function receives as an argument the size (in Gb) of the file to be created
> - The returned value of the function is saved in the `success` variable and then checked
> - Read the definition of the function in `../../A1_auxiliaryScripts/ofContainersOverlayFunctions.sh`
>
> <p>&nbsp;</p>
> 
> ~~~
> #7. Replicating the ./overlayFSDir/overlay0 file into the needed number of ./overlayFSDir/overlay* files (as many as processors*)
> echo "Replication ./overlayFSDir/overlay0 into the rest of the ./overlayFSDir/overlay* files"
> for ii in $(seq 1 $(( foam_numberOfSubdomains - 1 ))); do
>     if [ -f ./overlayFSDir/overlay${ii} ]; then
>        echo "./overlayFSDir/overlay${ii} already exists"
>        echo "Deal with it first and remove it from the working directory"
>        echo "Exiting";exit 1
>     else
>        echo "Replicating ./overlayFSDir/overlay0 into ./overlayFSDir/overlay${ii}"
>        srun -n 1 -N 1 --mem-per-cpu=0 --exclusive cp ./overlayFSDir/overlay0 ./overlayFSDir/overlay${ii} &
>     fi
> done
> wait
> ~~~
> {: .language-bash}
> 
> ~~~
> #8. Creating the processor* directories inside the ./overlayFSDir/overlay* files
> createInsideProcessorDirs $insideDir $foam_numberOfSubdomains;success=$? #Calling the function for creatingthe inside directories 
> if [ $success -eq 222 ]; then 
>    echo "$insideDir/processor0 already exists inside the ./overlayFSDir/overlay0 file"
>    echo "Exiting";exit 1
> elif [ $success -ne 0 ]; then 
>    echo "Failed creating the inside directories, exiting"
>    echo "Exiting";exit 1
> fi
> ~~~
> {: .language-bash}
> - Note the use of the `createInsideProcessorDirs` function
> - The function receives as arguments: 1) the path inside the OverlayFS files where to create the `processor*` dirs, and 2) the number of subdomains.
> - The returned value of the function is saved in the `success` variable and then checked
> - Read the definition of the function in `../../A1_auxiliaryScripts/ofContainersOverlayFunctions.sh`
>
> <p>&nbsp;</p>
>
> ~~~
> #9. Transfer the content of the ./bakDir/bak.processor* directories into the ./overlayFSDir/overlay* files
> echo "Copying OpenFOAM the files inside ./bakDir/bak.processor* into the ./overlayFSDir/overlay* files"
> copyIntoOverlayII './bakDir/bak.processor${ii}/*' "$insideDir/"'processor${ii}/' "$foam_numberOfSubdomains" "true";success=$? 
> if [ $success -ne 0 ]; then 
>    echo "Failed creating the inside directories, exiting"
>    echo "Exiting";exit 1
> fi
> ~~~
> {: .language-bash}
> - Note the use of the `copyIntoOverlayII` function
> - The function receives as arguments: 1) the source of the copy, 2) the destination, 3) the number of subdomains and 4) a boolean for replacing content or not
> - Note the use of single quotes for passing the wildcard '*' to the function without evaluation
> - Also note the use of single quotes '...${ii}...' in the place where the number of the overlay${ii} (or processor${ii}) is needed
> - The returned value of the function is saved in the `success` variable and then checked
> - Read the definition of the function in `../../A1_auxiliaryScripts/ofContainersOverlayFunctions.sh`
>
> <p>&nbsp;</p>
>
> ~~~
> #10. Mark the initial conditions time directory as already fully reconstructed
> echo "Marking the time directory \"0\" as fully reconstructed"
> touch 0/.reconstructDone
> ~~~
> {: .language-bash}
> - A dummy empty and hidden file named `.reconstructDone` is used to mark those result-times that have been successfully reconstructed
> - In this case, as the time `0` is originally reconstructed by default, it is marked
>
> <p>&nbsp;</p>
>
> ~~~
> #11. List the content of directories inside the ./overlayFSDir/overlay* files
> echo "Listing the content in ./overlayFSDir/overlay0 $insideDir/processor0"
> srun -n 1 -N 1 singularity exec --overlay ./overlayFSDir/overlay0 $theImage ls -lat $insideDir/processor0/
> ~~~
> {: .language-bash}
>
{: .solution}

> ## C.I Steps for dealing with the Overlay setup
> 1. Submit the solver script (from the scripts directory)
> 
>    ~~~
>    zeus-1:*-2.4.x> sbatch --reservation=$myReservation C.prepareOverlayFoam.sh 
>    ~~~
>    {: .bash}
>    - If a reservation is not available, do not use the option. (Or you can use the debugq: `--partition=debugq` instead.)
>    
>    ~~~
>    Submitted batch job 4642685 on cluster zeus
>    ~~~
>    {: .output}
> 
> 2. Check that the processor* directories were renamed/moved to ./bakDir/bak.processor*:
> 
>    ~~~
>    zeus-1:*-2.4.x> cd run/channel395
>    zeus-1:channel395> ls ./bakDir
>    ~~~
>    {: .bash}
> 
>    ~~~
>    bak.processor0  bak.processor1  bak.processor2  bak.processor3  bak.processor4
>    ~~~
>    {: .output}
>    - All `processor*` directories have been renamed/moved to `./bakDir/bak.processor*`
>
> 3. Check that ./overlayFSDir/overlay* files exist:
> 
>    ~~~
>    zeus-1:channel395> ls ./overlayFSDir
>    ~~~
>    {: .bash}
> 
>    ~~~
>    overlay0  overlay1  overlay2  overlay3  overlay4
>    ~~~
>    {: .output}
>    - All `./overlayFSDir/overlay*` files exist
>
> 4. Explore the content of one of the overlay files:
>
>    ~~~
>    zeus-1:channel395> module load singularity
>    zeus-1:channel395> theImage=/group/singularity/pawseyRepository/OpenFOAM/openfoam-2.4.x-pawsey.sif
>    zeus-1:channel395> insideDir=/overlayOpenFOAM/run/channel395
>    zeus-1:channel395> singularity exec --overlay ./overlayFSDir/overlay1 $theImage ls -lat $insideDir/processor1/
>    ~~~
>    {: .bash}
>
>    ~~~
>    total 16
>    drwxr-s---+ 4 espinosa pawsey0001 4096 May 24 20:38 .
>    drwxr-s---+ 2 espinosa pawsey0001 4096 May 24 20:38 0
>    drwxr-s---+ 3 espinosa pawsey0001 4096 May 24 20:38 constant
>    drwxr-s---+ 3 espinosa pawsey0001 4096 May 24 20:38 ..
>    ~~~
>    {: .output}
{: .discussion}

<p>&nbsp;</p>

## D. Executing the solver
> ## The `D.runFoam` script (main points to be discussed):
> ~~~
> #SBATCH --ntasks=5
> ~~~
> {: .language-bash}
> 
> ~~~
> #5. Defining OpenFOAM controlDict settings for this run
> foam_endTime=40
> foam_writeInterval=1
> foam_purgeWrite=0 #Just for testing in this exercise. In reality this should have a reasonable value if possible
> #foam_purgeWrite=10 #Just 10 times will be preserved
> ~~~
> {: .language-bash}
> 
> ~~~
> #7. Creating soft links towards directories inside the ./overlayFSDir/overlay* files
> #These links and directories will be recognized by each mpi instance of the container
> #(Initially these links will appear broken as they are pointing towards the interior of the ./overlayFSDir/overlay* files.
> # They will only be recognized within the containers)
> pointToOverlay $insideDir $foam_numberOfSubdomains;success=$? #Calling function to point towards the interior
> if [ $success -ne 0 ]; then
>    echo "Failed creating the soft links"
>    echo "Exiting";exit 1
> fi
> ~~~
> {: .language-bash}
> - Note the use of the `pointToOverlay` function
> - The function receives as arguments: 1) the path inside the OverlayFS files where to create the `processor*` dirs, and 2) the number of subdomains.
> - The returned value of the function is saved in the `success` variable and then checked
> - Read the definition of the function in `../../A1_auxiliaryScripts/ofContainersOverlayFunctions.sh`
>
> <p>&nbsp;</p>
>
> ~~~
> #8. Execute the case 
> echo "About to execute the case"
> srun -n $SLURM_NTASKS -N $SLURM_JOB_NUM_NODES bash -c "singularity exec --overlay ./overlayFSDir/"'overlay${SLURM_PROCID}'" $theImage pimpleFoam -parallel 2>&1" | tee $logsDir/log.pimpleFoam.$SLURM_JOBID
> echo "Execution finished"
> ~~~
> {: .language-bash}
> - VERY IMPORTANT: Note that the singularity command is called inside a `bash -c` command
> - This is the way we allow each MPI task to pick a different `./overlayFSDir/overlay*` file through the `SLURM_PROCID` variable
> - Here, `SLURM_PROCID` is slurm environment variable which needs to be evaluated when executing the container, so we use the section in single quotes `'...'` to allow the internal evaluation of that variable
> - Here, `theImage` is not a global environment variable, is evaluated by the host shell in a section with double quotes `"..."` at the command line
> - The total string passed to `bash -c` is the concatenation of two doble quotes sections with a single quotes section in between
> 
> <p>&nbsp;</p>
>
> ~~~
> #10. List the existing times inside the ./overlayFSDir/overlay0 
> echo "Listing the available times inside ./overlayFSDir/overlay0"
> srun -n 1 -N 1 singularity exec --overlay ./overlayFSDir/overlay0 $theImage ls -lat processor0/
> ~~~
> {: .language-bash}
{: .solution}

> ## D.I Steps for dealing with the solver
> 1. Submit the solver script (from the scripts directory)
> 
>    ~~~
>    zeus-1:*-2.4.x> sbatch --reservation=$myReservation D.runFoam.sh 
>    ~~~
>    {: .bash}
>    - If a reservation is not available, do not use the option. (Or you can use the debugq: `--partition=debugq` instead.)
>    
>    ~~~
>    Submitted batch job 4632685 on cluster zeus
>    ~~~
>    {: .output}
> 
> 2. Check that the solver is running:
> 
>    ~~~
>    zeus-1:*-2.4.x> squeue -u $USER
>    ~~~
>    {: .bash}
> 
>    ~~~
>    JOBID    USER     ACCOUNT     PARTITION            NAME EXEC_HOST ST     REASON   START_TIME     END_TIME  TIME_LEFT NODES   PRIORITY
>    4632685  espinosa pawsey0001  workq        D.runFoam.sh       n/a PD  Resources     17:09:28     17:19:28      10:00     1      75190 
>    ~~~
>    {: .output}
> 
>    Observe the output of the job with `tail -f` at runtime (use `<Ctrl-C>` to exit the command):
>    ~~~
>    zeus-1:*-2.4.x> tail -f slurm-4632685.out
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
> 
> 3. You can see in the case directory that now there are several `processor*` **soft links**
>    ~~~
>    zeus-1:*-2.4.x> cd run/channel395
>    zeus-1:channel395> ls -lat 
>    ~~~
>    {: .bash}
>
>    ~~~
>    total 5242952
>    drwxr-s---+ 12 espinosa pawsey0001       4096 May 25 14:55 .
>    drwxrws---+  4 espinosa pawsey0001       4096 May 25 14:55 logs
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor0 -> /overlayOpenFOAM/run/channel395/processor0
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor1 -> /overlayOpenFOAM/run/channel395/processor1
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor2 -> /overlayOpenFOAM/run/channel395/processor2
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor3 -> /overlayOpenFOAM/run/channel395/processor3
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor4 -> /overlayOpenFOAM/run/channel395/processor4
>    drwxr-s---+  2 espinosa pawsey0001       4096 May 25 14:55 system
>    drwxrws---+  4 espinosa pawsey0001       4096 May 25 14:53 bakDir
>    drwxrws---+  4 espinosa pawsey0001       4096 May 25 14:53 overlayFSDir
>    drwxr-s---+  2 espinosa pawsey0001       4096 May 25 14:53 0
>    drwxr-s---+  3 espinosa pawsey0001       4096 May 25 14:53 constant
>    drwxrws---+  3 espinosa pawsey0001       4096 May 25 14:03 ..
>    drwxr-s---+  2 espinosa pawsey0001       4096 May 25 14:03 0.org
>    -rwxrwx---+  1 espinosa pawsey0001        483 May 25 14:03 Allrun
>    ~~~
>    {: .output}
>    - The `processor*` soft links are pointing to the directory structure that "lives" inside the `./overlayFSDir/overlay*` files
>
>    
>    ~~~
>    zeus-1:channel395> ls -la processor1/ 
>    ~~~
>    {: .bash}
>
>    ~~~
>    ls: cannot access 'processor1/': No such file or directory
>    ~~~
>    {: .output}
>    - The host shell cannot read the directory structure inside the ./overlayFSDir/overlay* files, and that is why the links appear broken 
>
> 4. Check that the solver gave some results by listing the interior of an overlay file:
> 
>    ~~~
>    zeus-1:channel395> module load singularity
>    zeus-1:channel395> theImage=/group/singularity/pawseyRepository/OpenFOAM/openfoam-2.4.x-pawsey.sif
>    zeus-1:channel395> singularity exec --overlay ./overlayFSDir/overlay1 $theImage ls processor1/
>    ~~~
>    {: .bash}
> 
>    ~~~
>    0    10.2  12.4  14.6  16.8  19    20.2  22.4  24.6  26.8  29	 30.2  32.4  34.6  36.8  39    5    7.2  9.4
>    0.2  10.4  12.6  14.8  17    19.2  20.4  22.6  24.8  27    29.2  30.4  32.6  34.8  37	 39.2  5.2  7.4  9.6
>    0.4  10.6  12.8  15    17.2  19.4  20.6  22.8  25    27.2  29.4  30.6  32.8  35    37.2  39.4  5.4  7.6  9.8
>    0.6  10.8  13	 15.2  17.4  19.6  20.8  23    25.2  27.4  29.6  30.8  33    35.2  37.4  39.6  5.6  7.8  constant
>    0.8  11    13.2  15.4  17.6  19.8  21	 23.2  25.4  27.6  29.8  31    33.2  35.4  37.6  39.8  5.8  8
>    1    11.2  13.4  15.6  17.8  2	   21.2  23.4  25.6  27.8  3	 31.2  33.4  35.6  37.8  4     6    8.2
>    1.2  11.4  13.6  15.8  18    2.2   21.4  23.6  25.8  28    3.2	 31.4  33.6  35.8  38	 4.2   6.2  8.4
>    1.4  11.6  13.8  16    18.2  2.4   21.6  23.8  26    28.2  3.4	 31.6  33.8  36    38.2  4.4   6.4  8.6
>    1.6  11.8  14	 16.2  18.4  2.6   21.8  24    26.2  28.4  3.6	 31.8  34    36.2  38.4  4.6   6.6  8.8
>    1.8  12    14.2  16.4  18.6  2.8   22	 24.2  26.4  28.6  3.8	 32    34.2  36.4  38.6  4.8   6.8  9
>    10   12.2  14.4  16.6  18.8  20    22.2  24.4  26.6  28.8  30	 32.2  34.4  36.6  38.8  40    7    9.2
>    ~~~
>    {: .output}
>    - (In this example the final time was set to be 40 to allow for the creation of more results)
>
>    <p>&nbsp;</p>
>
> 5. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/run/`
{: .challenge}

<p>&nbsp;</p>

## E. Reconstruction

> ## No, unfortunately a container cannot mount more than 1 OverlayFS file at the same time
> - Yes, this implies that the results need to be copied back to the host file system before reconstruction
> - This is the inverse operation to the process of copying the initial decomposition into the OverlayFS files (explained at the beginning of this episode)
> - But in order to avoid the presence of many files in the host, this should be done by small batches:
>    1. Copy small batch of results from the interior to the `bak.processor*` directories
>    2. Now create `processor*` soft links to point to `bak.processor*` directories and not to the OverlayFS interior
>    3. Reconstruct that small batch
>    4. Remove the reconstructed result-times from the `bak.processor*` directories
>    5. Continue the cycle in 1. again until postprocessing all the result-times needed
>
{: .prereq}

<p>&nbsp;</p>

> ## The `E.reconstructFromOverlay.sh` script (main points to be discussed):
> ~~~
> #SBATCH --ntasks=4 #Several tasks will be used for copying files. (Independent from the numberOfSubdomains)
> ~~~
> {: .language-bash}
>
> ~~~
> #4. Create the reconstruction array, intended times to be reconstructed are set with the reconstructTimes var
> #These formats are the only accepted by function "generateReconstructArray" (check the function definition for further information)
> #reconstructTimes="all"
> #reconstructTimes="-1"
> #reconstructTimes="20"
> #reconstructTimes="50,60,70,80,90"
> reconstructTimes="0:10"
> unset arrayReconstruct #This global variable will be re-created in the function below
> generateReconstructArray "$reconstructTimes" "$insideDir";success=$? #Calling fucntion to generate "arrayReconstruct"
> if [ $success -ne 0 ]; then
>    echo "Failed creating the arrayReconstruct"
>    echo "Exiting";exit 1
> fi
> ~~~
> {: .language-bash}
> - Note the use of the `generateReconstructArray` function
> - The function receives as arguments: 1)the string defining the result-times to be reconstructed, and 2) the path inside the OverlayFS files where to create the `processor*` dirs
> - The returned value of the function is saved in the `success` variable and then checked
> - The GLOBAL array `arrayReconstruct` is unset before calling the function, and the function will create a new one with the result-times to be reconstructed
> - Read the definition of the function in `../../A1_auxiliaryScripts/ofContainersOverlayFunctions.sh`
>
> <p>&nbsp;</p>
>
> 
> ~~~
> #5. Point the soft links to the ./bakDir/bak.processor* directories
> pointToBak $foam_numberOfSubdomains;success=$? #Calling function to point towards the bak.processors
> if [ $success -ne 0 ]; then
>    echo "Failed creating the soft links"
>    echo "Exiting";exit 1
> fi
> ~~~
> {: .language-bash}
> - Note the use of the `pointToBak` function
> - The function receives as argument the number of subdomains.
> - The returned value of the function is saved in the success variable and then checked
> - Read the definition of the function in `../../A1_auxiliaryScripts/ofContainersOverlayFunctions.sh`
>
> <p>&nbsp;</p>
>
> ~~~
> maxTimeTransfersFromOverlays=10
> ~~~
> {: .language-bash}
> - This variable sets the size of the batches to be postprocessed
> 
> <p>&nbsp;</p>
>
> - The following sections are executed inside a loop (not shown here) as many times needed to postprocess all the result-times required batch by batch:
>     ~~~
>     ## 9. Copy from the ./overlayFSDir/overlay* the full batch into ./bakDir/bak.processor*
>     unset arrayCopyIntoBak
>     arrayCopyIntoBak=("${hereToDoReconstruct[@]}")
>     replace="true"
>     copyResultsIntoBak "$insideDir" "$foam_numberOfSubdomains" "$replace" "${arrayCopyIntoBak[@]}";success=$? #Calling the function to copy time directories into bak.processor*
>     if [ $success -ne 0 ]; then
>        echo "Failed transferring files into bak.processor* directories"
>        echo "Exiting";exit 1
>     fi
>     ~~~
>     {: .language-bash}
>     - The array `hereToDoReconstruct` has the result-times to be processed in the current batch
>     - Note the use of the `copyResultsIntoBak` function
>     - The function receives as arguments: 1) the path inside the overlays, 2) the number of subdomains, 3) the indication for replacing or not already existing result-times in the bak directories and 4) the array with the result-times to process.
>     - The returned value of the function is saved in the success variable and then checked
>     - Read the definition of the function in `../../A1_auxiliaryScripts/ofContainersOverlayFunctions.sh`
>    
>     <p>&nbsp;</p>
> 
>     
>     ~~~
>     ## 10. Reconstruct all times for this batch.
>     echo "Start reconstruction"
>     logFileHere=$logsDir/log.reconstructPar.${SLURM_JOBID}_${hereToDoReconstruct[0]}-${hereToDoReconstruct[-1]}
>     srun -n 1 -N 1 singularity exec $theImage reconstructPar -time ${timeString} 2>&1 | tee $logFileHere
>     ~~~
>     {: .language-bash}
>     - The command in the `srun` line executes the reconstruction of the batch
>     - The variable `timeString` has the list of the result-times to be reconstructed in each batch. This was set in a previous step (in step ## 8., not shown here)
>     - The output of the reconstruction is saved in the file `logFileHere` (this file needs to be checked when reconstruction errors happen)
> - For more details of the logic of the loop refer to the full script itself
>
{: .solution}

> ## E.I Steps for dealing with reconstruction:
> 
> 1. Submit the reconstruction script (from the scripts directory)
> 
>    ~~~
>    zeus-1:*-2.4.x> sbatch --reservation=$myReservation E.reconstructFromOverlay.sh 
>    ~~~
>    {: .bash}
>    - If a reservation is not available, do not use the option. (Or you can use the debugq: `--partition=debugq` instead.)
>    
>    ~~~
>    Submitted batch job 4632899 on cluster zeus
>    ~~~
>    {: .output}
> 
> 2. Check that the reconstruction is being performed:
> 
>    ~~~
>    zeus-1:*-2.4.x> cd run/channel395/
>    zeus-1:channel395> watch ls ./bakDir/bak.processor0/ 
>    ~~~
>    {: .bash}
>    - The command `watch` executes the `ls` of the content of `./bakDir/bak.processor0` every 2 seconds
> 
>     <p>&nbsp;</p>
>
>    ~~~
>    Every 2.0s: ls ./bakDir/bak.processor0/                                     Mon Jun 15 18:33:16 2020
>    
>    0
>    0.2
>    0.4
>    0.6
>    0.8
>    1
>    1.2
>    constant
>    ~~~
>    {: .output}
>    - You can watch the progress of the first batch being copied to the `./bakDir/bak.processor0` directory
>    - The same is happening for the resto of the `./bakDir/bak.processor*` directories
>    - In this case `maxTimeTransfersFromOverlays=10` (set within the script) is the size of the batches
>    - After the copy of the first batch is finished, those result-times will be reconstructed
>    - After succesful reconstruction, those result-times will be removed from the host file system 
>    
>     <p>&nbsp;</p>
>
>    ~~~
>    Every 2.0s: ls ./bakDir/bak.processor0/                                     Mon Jun 15 18:35:36 2020
>    
>    0
>    2.2
>    2.4
>    2.6
>    constant
>    ~~~
>    {: .output}
>    - A few minutes later, the second batch is being copied to the `bak.processor*` directories
>    - Note that the first batch of files have been removed already from the system
>    
>     <p>&nbsp;</p>
>
>    ~~~
>    Every 2.0s: ls ./bakDir/bak.processor0/                                    Mon Jun 15 18:39:16 2020
>    
>    0
>    10
>    constant
>    ~~~
>    {: .output}
>    - When finished, the earliest and the latest result-times were kept in the `bak.processor*` directories (although this can be modified within the scripts if desired)
>    
>     <p>&nbsp;</p>
>
>    ~~~
>    <CTRL>-C (to exit the watch command)
>    ~~~
>    {: .bash}
>
> 3. Check for the existence of the reconstructed times:
>    ~~~
>    zeus-1:channel395> ls
>    ~~~
>    {: .bash}
>
>    ~~~
>    0    0.org  1.6  2.6  3.6  4.6  5.6  6.6  7.6  8.6  9.6             bak.processor2  overlay0  processor0  system
>    0.2  1      1.8  2.8  3.8  4.8  5.8  6.8  7.8  8.8  9.8             bak.processor3  overlay1  processor1
>    0.4  10     2    3    4    5    6    7    8    9    Allrun          bak.processor4  overlay2  processor2
>    0.6  1.2    2.2  3.2  4.2  5.2  6.2  7.2  8.2  9.2  bak.processor0  constant        overlay3  processor3
>    0.8  1.4    2.4  3.4  4.4  5.4  6.4  7.4  8.4  9.4  bak.processor1  logs            overlay4  processor4
>    ~~~
>    {: .output}
>
> 4. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/post/` (check the command for the `reconstructPar` command for understanding the naming convention of the log files. In this case, they contain the `SLURM_JOBID` and the `first-last` result-times reconstructed in each batch.
{: .challenge}

<p>&nbsp;</p>

## F. Extract from Overlay into Bak; and G. Reconstruct From Bak

> ## F.G.I These scripts are left for the user to try by themselves
> - `F.extractFromOverlayIntoBak.sh` is for extracting a batch of result-times from the overlay files (no reconstruction)
> - `G.reconstructFromBak.sh` is for reconstructing existing results in the `bak.processor*` directories
>
{: .challenge}

## Z. Further notes on how to use OpenFOAM and OpenFOAM containers at Pawsey

More on OverlayFS for singularity in: [https://sylabs.io/guides/3.5/user-guide/persistent_overlays.html](https://sylabs.io/guides/3.5/user-guide/persistent_overlays.html)

The usage of OpenFOAM and OpenFOAM containers at Pawsey has already been described in our documentation: [OpenFOAM documentation at Pawsey](https://support.pawsey.org.au/documentation/display/US/OpenFOAM)

and in a technical newsletter note: [https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04](https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04) 

<p>&nbsp;</p>
