---
title: "Executing the full workflow of a case"
teaching: 30
exercises: 30
questions:
- How can I use OpenFOAM containers at Pawsey supercomputers?
objectives:
- Explain how to use typical pre-processing, solving and post-processing OpenFOAM tools
keypoints:
- Use `singularity exec $image <OpenFOAM-Tool> <Tool-Options>` for using containerised OpenFOAM tools
- Pre- and Post-Processing are usually single threaded and should be executed on Zeus
- Always use the recommended Pawsey Best Practices for OpenFOAM
- Most recent versions of OpenFOAM are not installed system-wide at Pawsey's Supercomputers, but are available via singularity containers
---

## 0. Introduction

> ## Typical workflow
> - Typical steps for analysing a problem with OpenFOAM include:
>    - The setup of initial conditions, mesh definition and solver parameters
>    - Use of pre-processing tools (typically decompose the case into several subdomains)
>    - Execute a solver to obtain a flow field solution
>    - Use post-processing tools (typically reconstruct the decomposed results into a single domain result)
>
{: .prereq}

> ## 0.I Accessing the scripts for this episode
> - Here we make use of a series of scripts to cover a typical workflow in full.
> 
> 1. cd into the directory where the provided scripts are. In this case we'll use OpenFOAM-v1912.
> 
>     ~~~
>     zeus-1:~> cd $MYSCRATCH/pawseyTraining/containers-openfoam-workshop-scripts
>     zeus-1:*-scripts> cd 02_executingFullWorkflow/example_OpenFOAM-v1912
>     zeus-1:*-v1912> ls
>     ~~~
>     {: .bash}
>     
>     ~~~
>     A.extractTutorial.sh  B.adaptCase.sh  C.decomposeFoam.sh  D.runFoam.sh  E.reconstructFoam.sh  run
>     ~~~
>     {: .output}
>     
> 2. Quickly read one of the scripts, for example `C.decomposeFoam.sh`.
>     We recommed the following text readers:
>     - `view` (navigate with up and down arrows, use `:q` or `:q!` to quit)
>     - `less` (navigate with up and down arrows, use `q` to quit) (this one does not have syntax highlight)
>     - If you are using an editor to read the scripts, DO NOT MODIFY THEM! because your exercise could mess up
>     - (Try your own settings after succeding with the original exercise, ideally in a copy of the script)
>     
>     ~~~
>     zeus-1:*-v1912> view C.decomposeFoam.sh
>     ~
>     ~
>     ~
>     :q
>     zeus-1*-v1912>
>     ~~~
>     {: .bash}
{: .discussion}

> ## Sections and scripts for this episode
> - In the following sections, there are instructions for submitting these job scripts for execution in the supercomputer one by one:
>     - `A.extractTutorial.sh` **(already pre-executed)** is for copying a tutorial from the interior of the container into our local file system
>     - `B.adaptCase.sh` **(already pre-executed)** is for modifying the tutorial to comply with Pawsey's best practices 
>     - `C.decomposeFoam.sh` is for executing the mesher and the decomposition of initial condition into subdomains
>     - `D.runFoam.sh` is for executing the solver
>     - `E.reconstructFoam.sh` is for executing the reconstruction of the last available result time
>
{: .prereq}

> ## So how will this episode flow?
> - The first two scripts (A. and B.) have already been executed for you. So we can concentrate in the main three stages of OpenFOAM usage.
> - We'll start with a detailed explanation at the final step of section "B. Adapt the case".
> - And continue the explanation up to the beginning of section "C. Decomposition".
> - Users will then proceed by themselves afterwards.
> - At the end we'll discuss the main instructions in the scripts and the whole process.
>
{: .callout}

<p>&nbsp;</p>

## A. Extraction of the tutorial: channel395
   
> ## The `A.extractTutorial.sh` script 
> > ## Main command in the script:
> >
> > ~~~
> > tutorialCase=incompressible/pimpleFoam/LES/channel395
> > caseDir=$SLURM_SUBMIT_DIR/run/channel395
> > srun -n 1 -N 1 singularity exec $theImage cp -r /opt/OpenFOAM/OpenFOAM-$theVersion/tutorials/$tutorialCase $caseDir
> > ~~~
> > {: .language-bash}
> > 
> {: .callout}
> 
> > ## Other important parts of the script:
> >
> > ~~~
> > #!/bin/bash -l
> > #SBATCH --export=NONE
> > #SBATCH --time=00:05:00
> > #SBATCH --ntasks=1
> > #SBATCH --partition=copyq #Ideally, you should be using the copyq for this kind of processes
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #1. Load the necessary modules
> > module load singularity
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #2. Defining the container to be used
> > theRepo=/group/singularity/pawseyRepository/OpenFOAM
> > theContainerBaseName=openfoam
> > theVersion=v1912
> > theProvider=pawsey
> > theImage=$theRepo/$theContainerBaseName-$theVersion-$theProvider.sif
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #3. Defining the tutorial and the case directory
> > tutorialAppDir=incompressible/pimpleFoam/LES
> > tutorialName=channel395
> > tutorialCase=$tutorialAppDir/$tutorialName
> > 
> > baseWorkingDir=$SLURM_SUBMIT_DIR/run
> > if ! [ -d $baseWorkingDir ]; then
> >     echo "Creating baseWorkingDir=$baseWorkingDir"
> > mkdir -p $baseWorkingDir
> > fi
> > caseName=$tutorialName
> > caseDir=$baseWorkingDir/$caseName
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #4. Copy the tutorialCase to the workingDir
> > if ! [ -d $caseDir ]; then
> >    srun -n 1 -N 1 singularity exec $theImage cp -r /opt/OpenFOAM/OpenFOAM-$theVersion/tutorials/$tutorialCase $caseDir
> > else
> >    echo "The case=$caseDir already exists, no new copy has been performed"
> > fi
> > ~~~
> > {: .language-bash}
> > 
> {: .solution}
>
{: .solution}

> ## A.I Steps for dealing with the extraction of the "channel395" case: **- [Pre-Executed]**
> 1. Submit the job (no need for reservation as the script uses the `copyq` partition)
> 
>    ~~~
>    zeus-1:*-v1912> sbatch A.extractTutorial.sh 
>    ~~~
>    {: .bash}
> 
>    ~~~
>    Submitted batch job 4632458
>    ~~~
>    {: .output}
> 
>    ~~~
>    zeus-1:*-v1912> squeue -u $USER
>    ~~~
>    {: .bash}
> 
>    ~~~
>    JOBID    USER     ACCOUNT     PARTITION            NAME EXEC_HOST ST     REASON   START_TIME     END_TIME  TIME_LEFT NODES   PRIORITY
>    4632468  espinosa pawsey0001  workq      A.extractTutor       n/a PD       None          N/A          N/A       5:00     1      75190
>    ~~~
>    {: .output}
> 
> 3. Check that the tutorial has been copied to our local file system
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

## B. Adapt the case to your needs (and Pawsey best practices)

> ## The `B.adaptCase.sh` script
> > ## Main command in the script
> >
> > ~~~
> > foam_runTimeModifiable="false"
> > sed -i 's,^runTimeModifiable.*,runTimeModifiable    '"$foam_runTimeModifiable"';,' ./system/controlDict
> > ~~~
> > {: .language-bash}
> > 
> {: .callout}
> 
> > ## Other important parts of the script:
> >
> > ~~~
> > #5. Defining OpenFOAM controlDict settings for Pawsey Best Practices
> > ##5.1 Replacing writeFormat, runTimeModifiable and purgeRight settings
> > foam_writeFormat="binary"
> > sed -i 's,^writeFormat.*,writeFormat    '"$foam_writeFormat"';,' ./system/controlDict
> > foam_runTimeModifiable="false"
> > sed -i 's,^runTimeModifiable.*,runTimeModifiable    '"$foam_runTimeModifiable"';,' ./system/controlDict
> > foam_purgeWrite=10
> > sed -i 's,^purgeWrite.*,purgeWrite    '"$foam_purgeWrite"';,' ./system/controlDict
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > ##5.2 Defining the use of collated fileHandler of output results 
> > echo "OptimisationSwitches" >> ./system/controlDict
> > echo "{" >> ./system/controlDict
> > echo "   fileHandler collated;" >> ./system/controlDict
> > echo "}" >> ./system/controlDict
> > ~~~
> > {: .language-bash}
> {: .solution}
>
{: .solution}

> ## B.I Initial steps for dealing with the adaptation of the case **- [Pre-Executed]**
>
> 1. Submit the adaptation script
> 
>    ~~~
>    zeus-1:*-v1912> sbatch B.adaptCase.sh 
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Submitted batch job 4632548
>    ~~~
>    {: .output}
{: .solution} 

> ## B.II Final steps. To check the adapted settings:
>
> 1. cd into the case directory:
>
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
>    - Mesh and fluid properties definition are under the tree of the subdirectory `constant`
>    - Solver settings are in the "dictionaries" inside the `system` subdirectory 
>
> 2. Read the `controlDict` dictionary inside the `system` subdirectory:
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
{: .discussion}

<p>&nbsp;</p>

## C. Decomposition

> ## The `C.decomposeFoam.sh` script
> > ## Main command in the script:
> > ~~~
> > srun -n 1 -N 1 singularity exec $theImage decomposePar -cellDist -force
> > ~~~
> > {: .language-bash}
> >
> {: .callout}
> 
> > ## Other important parts of the script:
> >
> > ~~~
> > #!/bin/bash -l
> > #SBATCH --ntasks=1
> > #SBATCH --mem=4G
> > #SBATCH --ntasks-per-node=28
> > #SBATCH --clusters=zeus
> > #SBATCH --partition=workq
> > #SBATCH --time=0:10:00
> > #SBATCH --export=none
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #1. Load the necessary modules
> > module load singularity
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #2. Defining the container to be used
> > theRepo=/group/singularity/pawseyRepository/OpenFOAM
> > theContainerBaseName=openfoam
> > theVersion=v1912
> > theProvider=pawsey
> > theImage=$theRepo/$theContainerBaseName-$theVersion-$theProvider.sif
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > #3. Defining the case directory
> > baseWorkingDir=$SLURM_SUBMIT_DIR/run
> > caseName=channel395
> > caseDir=$baseWorkingDir/$caseName
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #4. Going into the case and creating the logs directory
> > if [ -d $caseDir ]; then
> >    cd $caseDir
> >    echo "pwd=$(pwd)"
> > else
> >    echo "For some reason, the case=$caseDir, does not exist"
> >    echo "Exiting"; exit 1
> > fi
> > logsDir=./logs/pre
> > if ! [ -d $logsDir ]; then
> >    mkdir -p $logsDir
> > fi
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #6. Defining the ioRanks for collating I/O
> > # groups of 2 for this exercise (please read our documentation for the recommendations for production runs)
> > export FOAM_IORANKS='(0 2 4 6)'
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #7. Perform all preprocessing OpenFOAM steps up to decomposition
> > echo "Executing blockMesh"
> > srun -n 1 -N 1 singularity exec $theImage blockMesh 2>&1 | tee $logsDir/log.blockMesh.$SLURM_JOBID
> > echo "Executing decomposePar"
> > srun -n 1 -N 1 singularity exec $theImage decomposePar -cellDist -force 2>&1 | tee $logsDir/log.decomposePar.$SLURM_JOBID
> > ~~~
> > {: .language-bash}
> {: .solution}
>
{: .solution}

> ## C.I Steps for dealing with decomposition:
> 
> 1. Submit the decomposition script from the scripts directory (use the reservation for the workshop if available)
>
>    ~~~
>    zeus-1:*-v1912> myReservation=containers
>    ~~~
>    {: .bash}
>
>    ~~~
>    zeus-1:*-v1912> sbatch --reservation=$myReservation C.decomposeFoam.sh 
>    ~~~
>    {: .bash}
>    
>    > ## If you do not have a reservation
>    > Then, submit normally (or choose the best partition for executing the exercise, the `debugq` for example:)
>    >    ~~~
>    >    zeus-1:*-v1912> sbatch -p debugq C.decomposeFoam.sh
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
{: .discussion}

<p>&nbsp;</p>

## D. Executing the solver

> ## The `D.runFoam.sh` script 
> > ## Main command in the script:
> >
> > ~~~
> > theSolver=myPimpleFoam
> > srun -n $SLURM_NTASKS -N $SLURM_JOB_NUM_NODES singularity exec $theImage $theSolver -parallel
> > ~~~
> > {: .language-bash}
> >
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
> > theSolver=pimpleFoam
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #11. Execute the case 
> > echo "About to execute the case"
> > srun -n $SLURM_NTASKS -N $SLURM_JOB_NUM_NODES singularity exec $theImage $theSolver -parallel 2>&1 | tee $logsDir/log.$theSolver.$SLURM_JOBID
> > echo "Execution finished"
> > ~~~
> > {: .language-bash}
> {: .solution}
>
{: .solution}

> ## D.I Steps for dealing with the solver
> 1. Submit the solver script (from the scripts directory)
> 
>    ~~~
>    zeus-1:*-v1912> sbatch --reservation=$myReservation D.runFoam.sh 
>    ~~~
>    {: .bash}
>    - If a reservation is not available, do not use the option. (Or you can use the debugq: `--partition=debugq` instead.)
>    
>    ~~~
>    Submitted batch job 4632685 on cluster zeus
>    ~~~
>    {: .output}
> 
> 2. Check that the status of your job:
> 
>    ~~~
>    zeus-1:*-v1912> squeue -u $USER
>    ~~~
>    {: .bash}
> 
>    ~~~
>    JOBID    USER     ACCOUNT     PARTITION            NAME EXEC_HOST ST     REASON   START_TIME     END_TIME  TIME_LEFT NODES   PRIORITY
>    4632685  espinosa pawsey0001  workq        D.runFoam.sh       n/a PD  Resources     17:09:28     17:19:28      10:00     1      75190 
>    ~~~
>    {: .output}
> 
> 3. Observe the output of the job with `tail -f` at runtime (press `<Ctrl-C>` to exit the command):
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
> 
> 4. Check that the solver gave some results:
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
> 5. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/run/`
{: .challenge}

<p>&nbsp;</p>

## E. Reconstruction

> ## The `E.reconstructFoam.sh` script 
> > ## Main command in the script:
> >
> > ~~~
> > srun -n 1 -N 1 singularity exec $theImage reconstructPar -latestTime
> > ~~~
> > {: .language-bash}
> >
> {: .callout}
> 
> > ## Other important parts of the script:
> > ~~~
> > #SBATCH --ntasks=1
> > #SBATCH --mem=16G
> > #SBATCH --ntasks-per-node=28
> > #SBATCH --clusters=zeus
> > ~~~
> > {: .language-bash}
> > 
> > ~~~
> > #7. Execute reconstruction
> > echo "Start reconstruction"
> > srun -n 1 -N 1 singularity exec $theImage reconstructPar -latestTime 2>&1 | tee $logsDir/log.reconstructPar.$SLURM_JOBID
> > ~~~
> > {: .language-bash}
> {: .solution}
>
{: .solution}

> ## E.1 Steps for dealing with reconstruction:
> 
> 1. Submit the reconstruction script (from the scripts directory)
> 
>    ~~~
>    zeus-1:*-v1912> sbatch --reservation=$myReservation E.reconstructFoam.sh 
>    ~~~
>    {: .bash}
>    - If a reservation is not available, do not use the option. (Or you can use the debugq: `--partition=debugq` instead.)
>    
>    ~~~
>    Submitted batch job 4632899 on cluster zeus
>    ~~~
>    {: .output}
> 
> 2. Check that the reconstruction has been performed (a directory for the last time of the solution, `10` in this case, should appear at in the case directory):
> 
>    ~~~
>    zeus-1:*-v1912> ls ./run/channel395/
>    ~~~
>    {: .bash}
> 
>    ~~~
>    0  0.orig  10  Allrun  constant  logs  processors4_0-1  processors4_2-3  system
>    ~~~
>    {: .output}
>
> 3. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/post/`
{: .challenge}

<p>&nbsp;</p>

## Y. The collated option
>
> ## Information about the `fileHandler collated` option
> 
> The default for common OpenFOAM installations outside Pawsey is to output results into many processor* subdirectories. 
> Indeed, as many as decomposed subdomains. So, for example, in a common installation you would have finished with the following directories in your `$caseDir`:
> 
> ~~~
> 0  0.orig  Allrun  constant  logs  processor0  processor1  processor2  processor3  system
> ~~~
> {: .output}
> 
> So, the default fileHandler for common installations is **uncollated**.
> That is not a problem for a small case like this tutorial, but it is indeed a problem when the decomposition requires hundreds of subdomains.
> Then if results are saved very frequently, and the user needs to keep all their output for analysis (purgeWrite=0), they can end with millions of files.
> And the problem increases if the user needs to analyse and keep results for multiple cases.
> 
> The existence of millions of files is already a problem for the user, but is also a problem for all our user base, because the file system manager gets overloaded and the performance of the file system (which is shared) degrades.
> 
> Because of that, for versions greater-than or equal-to OpenFOAM-6 and OpenFOAM-v1812, the use "**fileHandler collated**" is a required policy in our systems.
> (Older versions do not have that option or do not perform well.)
> The collated option allows result files to be merged into a reduce number of directories.
> (We have set that in point B.adaptCase (above) within the controlDict.
> Pawsey containers and system-wide installations have the collated option set by default.
> Nevertheless, we decided to explicitly include it in the controlDict as a reminder to our users.)
> 
> Together with the setting of the collated option, the merging of results is controlled by the **ioRanks** option.
> In this case, the option is being set through the **FOAM_IORANKS** environment variable.
>
> In the scripts we have:
> 
> ~~~
> export FOAM_IORANKS='(0 2 4 6 8)'
> ~~~
> {: .bash}
> which indicates that decomposed results will be merged every 2 subdomains. [0-1] in the first subdirectory, and [2-3] in the second one. As seen in our case directory:
> 
> ~~~
> 0  0.orig  Allrun  constant  logs  processors4_0-1  processors4_2-3  system
> ~~~
> {: .output}
> (The extra numbers in the list does not affect the settings, but strictly speaking '(0 2)' would be enough.) The "export" is needed for the variable to be exported to the srun executions.
> 
> If the ioRanks option were not set, then the collated option alone would have merged all the results into a single directory, as this:
> 
> ~~~
> 0  0.orig  Allrun  constant  logs  processors4  system
> ~~~
> {: .output}
> 
> In production runs, we recommend to group results for each node into a single directory. So, for a case to be solved on Magnus, the recommended setting is:
> 
> ~~~
> export FOAM_IORANKS='(0 24 48 72 . . . numberOfSubdomains)'
> ~~~
> {: .bash}
>
> and for a case to be solved on Zeus:
> 
> ~~~
> export FOAM_IORANKS='(0 28 56 84 . . . numberOfSubdomains)'
> ~~~
> {: .bash}
>
> If the numberOfSubdomains fit in a single node, we recommend to use the plain collated option alone without ioRanks.
> 
> Also, we recommend to decompose your domain so that subdomains count with ~100,000 cells. This rule of thumb gives good performance in most of situations, although users need to verify the best settings for their own cases and solvers.
> 
{: .solution}

<p>&nbsp;</p>

## Z. Further notes on how to use OpenFOAM and OpenFOAM containers at Pawsey

The usage of OpenFOAM and OpenFOAM containers at Pawsey has already been described in our documentation: [OpenFOAM documentation at Pawsey](https://support.pawsey.org.au/documentation/display/US/OpenFOAM)

and in a technical newsletter note: [https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04](https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04) 

<p>&nbsp;</p>
