---
title: "Use OverlayFS to reduce the number of result files"
teaching: 30
exercises: 30
questions:
- For old versions of OpenFOAM (without "collated" option available), how can I reduce the amount of result files?
objectives:
- Use OverlayFS for saving results and reduce the number of files in the host file system
keypoints:
- Singularity can deal with an OverlayFS, but only one OverlayFS can be mounted per container instance
- As each core writes results to a single `processorN`, this works for saving results inside each `overlayN`
- Unfortunately, the `reconstructPar` tool cannot read results from several `overlayN` files. Therefore, results must be taken out before reconstruction.
- Last point may seem like a killer, but extraction and reconstruction may be performed in small batches avoiding the appearence of many files at the same time for the Meta Data Server.
---

Due to multiple factors, users may have the need to keep using old versions of OpenFOAM.

Versions of OpenFOAM lower-than OpenFOAM-6 and OpenFOAM-v1812 do not count with a functional `fileHandler collated` + `ioRanks` option.
Therefore, there is not other option but the standard `uncollated` fileHandling.

As metioned previously, standard uncollated fileHandling has the danger of producing a large amount of files.
It is not uncommon for some of our users to reach the order of millions of files.

When users hit the presence of a large amount of files, they start facing problems for handling them.

But the main larger problem is the **overload of the metadata server** of our shared file systems, which affect the performance of the supercomputers themselves, affecting all of our user base.

Currently, Pawsey restricts the quota for the users to a maximum of 1 million files/directories (ioNodes). But many still have the need for a larger quota.

One option to reduce the amount of files that the system observes is the use of **virtual file systems**.

These virtual file systems are indeed a single file from the metadata server point of view. But the virtual file system is able to handle thousands of files at the interior.

Here we exemplify the use of OverlayFS virtual file systems (that singularity containers can read and write) to save OpenFOAM results inside.

In this way, the amount of files observed by our metadata server is reduced, while the user expands their capability of saving many result files (even in the order of millions). The example is managed with a series of scripts:

- cd into the directory where the provided scripts are. In this case we'll use OpenFOAM-2.4.x.

~~~
zeus-1:~> cd $MYSCRATCH/pawseyTraining/containers-openfoam-workshop-scripts
zeus-1:*-scripts> cd 05_useOverlayFSForReducingNumberOfFiles/example_OpenFOAM-2.4.x
zeus-1:*-2.4.x> ls
~~~
{: .bash}

~~~
A.extractAndAdaptTutorial.sh  caseSettingsFoam.sh      D.runFoam.sh                 imageSettingsSingularity.sh
B.decomposeFoam.sh            C.setupOverlayFoam.sh    E.reconstructFromOverlay.sh  run
~~~
{: .output}

Quickly read one of the scripts, for example `E.reconstructFromOverlay.sh`.
We recommed the following text readers:
- `view` (navigate with up and down arrows, use `:q` or `:q!` to quit)
- `less` (navigate with up and down arrows, use `q` to quit) (this one does not have syntax highlight)

~~~
zeus-1:*-2.4.x> view D.runFoam.sh
~
~
~
:q
zeus-1*-2.4.x>
~~~
{: .bash}

In the following sections, there are instructions for submitting these job scripts for execution in the supercomputer one by one. 

Very similar submission steps are executed in each of the stages.
So, in order to avoid too much repetition and save time for important discussions,
the section "A. Extract and adapt tutorial" has already been executed for you.

We'll start our explanation at section "B.Decomposition" but concentrate our efforts on section "C. Setup OverlayFS".
Users will then move to section D. and proceed by themselves afterwards.

At the end, we'll discuss the main instructions within the scripts and the whole process.

<p>&nbsp;</p>

## A. Extract and adapt the tutorial to be solved **- [Pre-Executed]**
   
> ## The `A.extractAndAdaptTutorial.sh` script (main sections to be discussed):
>
> ~~~
> #1. Loading the container settings, case settings and auxiliary functions (order is important)
> source $SLURM_SUBMIT_DIR/imageSettingsSingularity.sh
> source $SLURM_SUBMIT_DIR/caseSettingsFoam.sh
> ~~~
> {: .bash}
>
> > ## The `imageSettingsSingularity.sh` script (main sections to be discussed):
> >
> > ~~~
> > #Module environment
> > module load singularity
> > ~~~
> > {: .bash}
> >
> > ~~~
> > #Defining the container to be used
> > theRepo=/group/singularity/pawseyRepository/OpenFOAM
> > theContainerBaseName=openfoam
> > theVersion=2.4.x
> > theProvider=pawsey
> > theImage=$theRepo/$theContainerBaseName-$theVersion-$theProvider.sif
> > ~~~
> > {: .bash}
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
> > {: .bash}
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
> > {: .bash}
> {: .solution}
{: .solution}

> ## Steps for dealing with the extraction and adaptation of the case to be solved
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
>    > ## The settings that were adapted in `..../run/channel395/system/controlDict` 
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
> ~~~
> {: .bash}
>
> > ## The `imageSettingsSingularity.sh` script (main sections to be discussed):
> >
> > ~~~
> > #Module environment
> > module load singularity
> > ~~~
> > {: .bash}
> >
> > ~~~
> > #Defining the container to be used
> > theRepo=/group/singularity/pawseyRepository/OpenFOAM
> > theContainerBaseName=openfoam
> > theVersion=2.4.x
> > theProvider=pawsey
> > theImage=$theRepo/$theContainerBaseName-$theVersion-$theProvider.sif
> > ~~~
> > {: .bash}
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
> > {: .bash}
> {: .solution} 
> 
> ~~~
> #7. Perform all preprocessing OpenFOAM steps up to decomposition
> srun -n 1 -N 1 singularity exec $theImage blockMesh 2>&1 | tee $logsDir/log.blockMesh
> srun -n 1 -N 1 singularity exec $theImage decomposePar -cellDist 2>&1 | tee $logsDir/log.decomposePar
> ~~~
> {: .bash}
{: .solution}

> ## Steps for dealing with decomposition:
> 
> 1. Submit the decomposition script from the scripts directory (use the reservation for the workshop if available)
>
>    ~~~
>    zeus-1:*-2.4.x> myReservation=XXX
>    ~~~
>    {: .bash}
> 
>    ~~~
>    zeus-1:*-2.4.x> sbatch --reservation=$myReservation B.decomposeFoam.sh 
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
>    - Note that one `processorN` directory was created per `numberOfSubdomains` (The number of subdomains is set in the `system/decomposeParDict` dictionary)
>    - Also note that this tutorial uses 5 subdomains (and 5 cores when executing the solver (below))
>
> 3. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/pre/`
{: .challenge}

<p>&nbsp;</p>

## C. Setup the overlayFS
> ## The `C.setupOverlayFoam.sh` script (main points to be discussed):
> ~~~
> #1. Loading the container settings, case settings and auxiliary functions (order is important)
> source $SLURM_SUBMIT_DIR/imageSettingsSingularity.sh
> source $SLURM_SUBMIT_DIR/caseSettingsFoam.sh
> ~~~
> {: .bash}
>
> > ## The `imageSettingsSingularity.sh` script (main sections to be discussed):
> > ~~~
> > #Module environment
> > module load singularity
> > ~~~
> > {: .bash}
> >
> > ~~~
> > #Defining the container to be used
> > theRepo=/group/singularity/pawseyRepository/OpenFOAM
> > theContainerBaseName=openfoam
> > theVersion=2.4.x
> > theProvider=pawsey
> > theImage=$theRepo/$theContainerBaseName-$theVersion-$theProvider.sif
> > ~~~
> > {: .bash}
> >
> > ~~~
> > #Defining settings for the OverlayFS
> > overlaySizeGb=1
> > ~~~
> > {: .bash}
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
> > {: .bash}
> > 
> > ~~~
> > #Defining the name of the directory inside the overlay* files at which results will be saved
> > baseInsideDir=/overlayOpenFOAM/run
> > insideName=$caseName
> > insideDir=$baseInsideDir/$insideName
> > ~~~
> > {: .bash}
> {: .solution}
>
> ~~~
> #4. Rename the processor* directories into bak.processor*
> #(OpenFOAM wont be able to see these directories)
> #(Access will be performed through soft links)
> echo "Renaming the processor directories"
> rename processor bak.processor processor*
> ~~~
> {: .bash}
> 
> ~~~
> #5. Creating a first overlay file (overlay0)
> #(Needs to use ubuntu:18.04 or higher to use the -d <root-dir> option to make them writable by simple users)
> echo "Creating the overlay0 file"
> echo "The size in Gb is overlaySizeGb=$overlaySizeGb"
> if [ $overlaySizeGb -gt 0 ]; then
>    countSize=$(( overlaySizeGb * 1024 * 1024 ))
>    srun -n 1 -N 1 singularity exec docker://ubuntu:18.04 bash -c " \
>         mkdir -p overlay_tmp/upper && \
>         dd if=/dev/zero of=overlay0 count=$countSize bs=1024 && \
>         mkfs.ext3 -d overlay_tmp overlay0 && rm -rf overlay_tmp \
>         "
> else
>    echo "Variable overlaySizeGb was not set correctly"
>    echo "In theory, this should have been set together with the singularity settings"
>    echo "Exiting";exit 1
> fi
> ~~~
> {: .bash}
> 
> ~~~
> #6. Replicating the overlay0 file into the needed number of overlay* files (as many as processors*)
> echo "Replication overlay0 into the rest of the overlay* files"
> for ii in $(seq 1 $(( foam_numberOfSubdomains - 1 ))); do
>    if [ -f overlay${ii} ]; then
>       echo "overlay${ii} already exists"
>       echo "Deal with it first and remove it from the working directory"
>       echo "Exiting";exit 1
>    else
>       echo "Replicating overlay0 into overlay${ii}"
>       srun -n 1 -N 1 --mem-per-cpu=0 --exclusive cp overlay0 overlay${ii} &
>    fi
> done
> wait
> ~~~
> {: .bash}
> 
> ~~~
> #7. Creating inside processor* directories inside the overlayFS 
> echo "Creating the directories inside the overlays"
> for ii in $(seq 0 $(( foam_numberOfSubdomains - 1 ))); do
>    echo "Creating processor${ii} inside overlay${ii}"
>    srun -n 1 -N 1 --mem-per-cpu=0 --exclusive singularity exec --overlay overlay${ii} $theImage mkdir -p $insideDir/processor${ii} &
> done
> wait
> ~~~
> {: .bash}
> 
> ~~~
> #8. Transfer the content of the bak.processor* directories into the overlayFS
> echo "Copying OpenFOAM files inside bak.processor* into the overlays"
> for ii in $(seq 0 $(( foam_numberOfSubdomains - 1 ))); do
>     echo "Writing into overlay${ii}"
>     srun -n 1 -N 1 --mem-per-cpu=0 --exclusive singularity exec --overlay overlay${ii} $theImage cp -r bak.processor${ii}/* $insideDir/processor${ii}/ &
> done
> wait
> ~~~
> {: .bash}
> 
> ~~~
> #9. List the content of directories inside the overlay* files
> echo "Listing the content in overlay0 $insideDir/processor0"
> srun -n 1 -N 1 singularity exec --overlay overlay0 $theImage ls -lat $insideDir/processor0/
> ~~~
> {: .bash}
{: .solution}

> ## Steps for dealing with the Overlay setup
> 1. Submit the solver script (from the scripts directory)
> 
>    ~~~
>    zeus-1:*-2.4.x> sbatch --reservation=$myReservation C.prepareOverlayFoam.sh 
>    ~~~
>    
>    ~~~
>    Submitted batch job 4642685 on cluster zeus
>    ~~~
>    {: .output}
> 
> 2. Check that the overlay files were created and the processor* directories renamed to bak.processor*:
> 
>    ~~~
>    zeus-1:*-2.4.x> cd run/channel395
>    zeus-1:channel395> ls
>    ~~~
>    {: .bash}
> 
>    ~~~
>    0      Allrun          bak.processor1  bak.processor3  constant  overlay0  overlay2  overlay4
>    0.org  bak.processor0  bak.processor2  bak.processor4  logs      overlay1  overlay3  system
>    ~~~
>    {: .output}
>    - There are now 5 `overlayN` files
>    - All `processorN` directories have been renamed to `bak.processorN`
>
> 3. Explore the content of one of the overlay files:
>
>    ~~~
>    zeus-1:channel395> module load singularity
>    zeus-1:channel395> theImage=/group/singularity/pawseyRepository/OpenFOAM/openfoam-2.4.x-pawsey.sif
>    zeus-1:channel395> insideDir=/overlayOpenFOAM/run/channel395
>    zeus-1:channel395> singularity exec --overlay overlay1 $theImage ls -lat $insideDir/processor1/
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
{: .challenge}

<p>&nbsp;</p>

## D. Executing the solver
> ## The `D.runFoam` script (main points to be discussed):
> ~~~
> #SBATCH --ntasks=5
> ~~~
> {: .bash}
> 
> ~~~
> #5. Defining OpenFOAM controlDict settings for this run
> foam_writeInterval=1
> foam_purgeWrite=0 #Just for testing in this exercise. In reality this should have a reasonable value if possible
> #foam_purgeWrite=10 #Just 10 times will be preserved
> ~~~
> {: .bash}
> 
> ~~~
> #7. Creating soft links towards directories inside the overlayFS files
> #These links and directories will be recognized by each mpi instance of the container
> #(Initially these links will appear broken as they are pointing towards the interior of the overlay* files.
> # They will only be recognized within the containers)
> echo "Creating the soft links to point towards the interior of the overlay files"
>
> for ii in $(seq 0 $(( foam_numberOfSubdomains -1 ))); do
>    echo "Linking to $insideDir/processor${ii} in overlay${ii}"
>    srun -n 1 -N 1 --mem-per-cpu=0 --exclusive ln -s $insideDir/processor${ii} processor${ii} &
> done
> wait
> ~~~
> {: .bash}
>
> ~~~
> #8. Execute the case using the softlinks to write inside the overlays
> echo "About to execute the case"
> srun -n $SLURM_NTASKS -N $SLURM_JOB_NUM_NODES bash -c 'singularity exec --overlay overlay${SLURM_PROCID} '"$theImage"' pimpleFoam -parallel 2>&1' | tee $logsDir/log.pimpleFoam.$SLURM_JOBID
> echo "Execution finished"
> ~~~
> {: .bash}
> - VERY IMPORTANT: Note that the singularity command is called inside a `bash -c` command
> - This is the way we allow each MPI task to pick a different overlay file through the `SLURM_PROCID` variable
> - Here, `theImage` is not a global environment variable, so we use the shift to a `"..."` section to evaluate the variable in the host shell
> 
> ~~~
> #9. List the existing times inside the overlays 
> echo "Listing the available times inside overlay0"
> srun -n 1 -N 1 singularity exec --overlay overlay0 $theImage ls -lat processor0/
> ~~~
> {: .bash}
{: .solution}

> ## Steps for dealing with the solver
> 1. Submit the solver script (from the scripts directory)
> 
>    ~~~
>    zeus-1:*-2.4.x> sbatch --reservation=$myReservation D.runFoam.sh 
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
> 3. You can see in the case directory that now there are several `processorN` **soft links**
>    ~~~
>    zeus-1:*-2.4.x> cd run/channel395
>    zeus-1:channel395> ls -lat 
>    ~~~
>    {: .bash}
>
>    ~~~
>    total 5242952
>    -rw-rw----+  1 espinosa pawsey0001 1073741824 May 25 14:55 overlay0
>    -rw-rw----+  1 espinosa pawsey0001 1073741824 May 25 14:55 overlay1
>    -rw-rw----+  1 espinosa pawsey0001 1073741824 May 25 14:55 overlay2
>    -rw-rw----+  1 espinosa pawsey0001 1073741824 May 25 14:55 overlay3
>    -rw-rw----+  1 espinosa pawsey0001 1073741824 May 25 14:55 overlay4
>    drwxr-s---+ 12 espinosa pawsey0001       4096 May 25 14:55 .
>    drwxrws---+  4 espinosa pawsey0001       4096 May 25 14:55 logs
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor0 -> /overlayOpenFOAM/run/channel395/processor0
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor1 -> /overlayOpenFOAM/run/channel395/processor1
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor2 -> /overlayOpenFOAM/run/channel395/processor2
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor3 -> /overlayOpenFOAM/run/channel395/processor3
>    lrwxrwxrwx   1 espinosa pawsey0001         42 May 25 14:55 processor4 -> /overlayOpenFOAM/run/channel395/processor4
>    drwxr-s---+  2 espinosa pawsey0001       4096 May 25 14:55 system
>    drwxrws---+  4 espinosa pawsey0001       4096 May 25 14:53 bak.processor0
>    drwxrws---+  4 espinosa pawsey0001       4096 May 25 14:53 bak.processor1
>    drwxrws---+  4 espinosa pawsey0001       4096 May 25 14:53 bak.processor2
>    drwxrws---+  4 espinosa pawsey0001       4096 May 25 14:53 bak.processor3
>    drwxrws---+  4 espinosa pawsey0001       4096 May 25 14:53 bak.processor4
>    drwxr-s---+  2 espinosa pawsey0001       4096 May 25 14:53 0
>    drwxr-s---+  3 espinosa pawsey0001       4096 May 25 14:53 constant
>    drwxrws---+  3 espinosa pawsey0001       4096 May 25 14:03 ..
>    drwxr-s---+  2 espinosa pawsey0001       4096 May 25 14:03 0.org
>    -rwxrwx---+  1 espinosa pawsey0001        483 May 25 14:03 Allrun
>    ~~~
>    {: .output}
>    - The `processorN` soft links are pointing to the directories inside the `overlayN` files
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
>    - The host shell cannot read inside the overlayN files, and that is why the links appear broken 
>
> 4. Check that the solver gave some results by listing the interior of an overlay file:
> 
>    ~~~
>    zeus-1:channel395> module load singularity
>    zeus-1:channel395> theImage=/group/singularity/pawseyRepository/OpenFOAM/openfoam-2.4.x-pawsey.sif
>    zeus-1:channel395> singularity exec --overlay overlay1 $theImage ls processor1/
>    ~~~
>    {: .bash}
> 
>    ~~~
>    0    0.6  1.2  1.8  2.2  2.8  3.4  4	4.6  5.2  5.8  6.4  7	 7.6  8.2  8.8	9.4  constant
>    0.2  0.8  1.4  10   2.4  3    3.6  4.2	4.8  5.4  6    6.6  7.2  7.8  8.4  9	9.6
>    0.4  1	  1.6  2    2.6  3.2  3.8  4.4	5    5.6  6.2  6.8  7.4  8    8.6  9.2	9.8
>    ~~~
>    {: .output}
>
> 5. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/run/`
{: .challenge}

<p>&nbsp;</p>

## E. Reconstruction
> ## The `E.reconstructFromOverlay.sh` script (main points to be discussed):
> ~~~
> #SBATCH --ntasks=4 #Independent from the number of subdomains
> ~~~
> {: .bash}
>
> ~~~
> #4. Transfer the content of the overlayFS into the bak.processor* directories using the softlinks processor*
> reconstructionTime=10
> echo "Copying the times to reconstruct from the overlays into bak.processor*"
> for ii in $(seq 0 $(( foam_numberOfSubdomains - 1 ))); do
>    echo "Writing into bak.processor${ii}"
>    srun -n 1 -N 1 --mem-per-cpu=0 --exclusive singularity exec --overlay overlay${ii} $theImage cp -r $insideDir/processor${ii}/$reconstructionTime bak.processor${ii} &
> done
> wait
> ~~~
> {: .bash}
> - To be able to reconstruct a specific time, information needs to be transferred again to the `bak.processorN` directories
> 
> ~~~
> #5. Point the soft links to the bak.processor* directories
> echo "Creating the soft links to point towards the bak.processor* directories"
> for ii in $(seq 0 $(( foam_numberOfSubdomains -1 ))); do
>    echo "Linking to bak.processor${ii}"
>    srun -n 1 -N 1 --mem-per-cpu=0 --exclusive ln -s bak.processor${ii} processor${ii} &
> done
> wait
> ~~~
> {: .bash}
> - Now new `processorN` soft links will point towards the `bak.processorN` physical directories
>
> ~~~
> #6. Reconstruct the indicated time
> echo "Start reconstruction"
> srun -n 1 -N 1 singularity exec $theImage reconstructPar -time ${reconstructionTime} 2>&1 | tee $logsDir/log.reconstructPar.$SLURM_JOBID
> if grep -i 'error\|exiting' $logsDir/log.reconstructPar.$SLURM_JOBID; then
>    echo "The reconstruction of time ${reconstructionTime} failed"
>    echo "Exiting";exit 1
> fi
> ~~~
> {: .bash}
{: .solution}

> ## Steps for dealing with reconstruction:
> 
> 1. Submit the reconstruction script (from the scripts directory)
> 
>    ~~~
>    zeus-1:*-2.4.x> sbatch --reservation=$myReservation E.reconstructFromOverlay.sh 
>    ~~~
>    {: .bash}
>    
>    ~~~
>    Submitted batch job 4632899 on cluster zeus
>    ~~~
>    {: .output}
> 
> 2. Check that the reconstruction has been performed (a directory for the last time of the solution, `10` in this case, should appear at in the case directory):
> 
>    ~~~
>    zeus-1:*-2.4.x> ls ./run/channel395/
>    ~~~
>    {: .bash}
> 
>    ~~~
>    0      Allrun          bak.processor2  constant  overlay1  overlay4    processor2  system
>    0.org  bak.processor0  bak.processor3  logs      overlay2  processor0  processor3
>    10     bak.processor1  bak.processor4  overlay0  overlay3  processor1  processor4
>    ~~~
>    {: .output}
>
> 3. You should also check for success/errors in:
>    -  the slurm output file: `slurm-<SLURM_JOBID>.out`
>    -  the log files created when executing the OpenFOAM tools in: `./run/channel395/logs/post/`
{: .challenge}

<p>&nbsp;</p>

## Z. Further notes on how to use OpenFOAM and OpenFOAM containers at Pawsey

More on OverlayFS for singularity in: [https://sylabs.io/guides/3.5/user-guide/persistent_overlays.html](https://sylabs.io/guides/3.5/user-guide/persistent_overlays.html)

The usage of OpenFOAM and OpenFOAM containers at Pawsey has already been described in our documentation: [OpenFOAM documentation at Pawsey](https://support.pawsey.org.au/documentation/display/US/OpenFOAM)

and in a technical newsletter note: [https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04](https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04) 

<p>&nbsp;</p>
