---
title: "Executing the full processing of a case"
teaching: 15
exercises: 30
questions:
objectives:
- Explain how to use typical pre-processing, solving and post-processing tools
keypoints:
- A
---

Typical steps for analasing a problem with OpenFOAM and include the use of pre-processing, solving and post-processing tools.
Here we make use of a series of scripts to cover a typical workflow in full.

0. cd into the directory where the provided scripts are. In this case we'll use OpenFOAM-v1912.

   ~~~
   zeus-1:~> cd $MYSCRATCH/pawseyTraining/containers-openfoam-workshop-scripts
   zeus-1:*-scripts> cd 02_executingFullProcess/example_OpenFOAM-v1912
   zeus-1:*-v1912> ls
   ~~~
   {: .bash}
   
   ~~~
   ~~~
   {: .output}


## A. Extraction of the tutorial (channel395)

2. Read the extraction script. (You can use `less` or `vi` or any text editor/reader of your preference)

   ~~~
   zeus-1:*-v1912> less A.extractTutorial.sh
   .
   .
   .
   q
   zeus-1*-v1912>
   ~~~
   {: .bash}
   
   > ## The `A.extractTutorial.sh` script (sections to explain):
   >
   > ~~~
   > #!/bin/bash -l
   > #SBATCH --export=NONE
   > #SBATCH --time=00:05:00
   > #SBATCH --ntasks=1
   > #@@#SBATCH --partition=copyq #Ideally, you should be using the copyq for this kind of processes
   > #SBATCH --partition=workq
   > ~~~
   > {: .bash}
   > 
   > ~~~
   > #1. Load the necessary modules
   > module load singularity
   > ~~~
   > {: .bash}
   > 
   > ~~~
   > #2. Defining the container to be used
   > theRepo=/group/singularity/pawseyRepository/OpenFOAM
   > theContainerBaseName=openfoam
   > theVersion=v1912
   > theProvider=pawsey
   > theImage=$theRepo/$theContainerBaseName-$theVersion-$theProvider.sif
   > ~~~
   > {: .bash}
   > 
   > ~~~
   > #3. Defining the tutorial and the case directory
   > tutorialAppDir=incompressible/pimpleFoam/LES
   > tutorialName=channel395
   > tutorialCase=$tutorialAppDir/$tutorialName
   > 
   > baseWorkingDir=./run
   > if ! [ -d $baseWorkingDir ]; then
   >     echo "Creating baseWorkingDir=$baseWorkingDir"
   > mkdir -p $baseWorkingDir
   > fi
   > caseName=$tutorialName
   > caseDir=$baseWorkingDir/$caseName
   > ~~~
   > {: .bash}
   > 
   > ~~~
   > #4. Copy the tutorialCase to the workingDir
   > if ! [ -d $caseDir ]; then
   >    #srun -n 1 -N 1 singularity exec $theImage bash -c 'cp -r $FOAM_TUTORIALS/'"$tutorialCase $caseDir"
   >    srun -n 1 -N 1 singularity exec $theImage cp -r /opt/OpenFOAM/OpenFOAM-$theVersion/tutorials/$tutorialCase $caseDir
   > else
   >    echo "The case=$caseDir already exists, no new copy has been performed"
   > fi
   > ~~~
   > {: .bash}
   > 
   {: .solution}

3. Submit the job using our reservation
   ~~~
   zeus-1:*-v1912> myReservation=XXX
   ~~~
   {: .bash}

   ~~~
   zeus-1:*-v1912> sbatch --reservation=$myReservation A.extractTutorial.sh 
   ~~~
   {: .bash}

   ~~~
   Submitted batch job 4632458
   ~~~
   {: .output}

   ~~~
   zeus-1:*-v1912> squeue -u espinosa
   ~~~
   {: .bash}

   ~~~
   JOBID    USER     ACCOUNT     PARTITION            NAME EXEC_HOST ST     REASON   START_TIME     END_TIME  TIME_LEFT NODES   PRIORITY
   4632468  espinosa pawsey0001  workq      A.extractTutor       n/a PD       None          N/A          N/A       5:00     1      75190
   ~~~
   {: .output}

4. Check that the tutorial has been copied to our local file system

   ~~~
   zeus-1:*-v1912> ls ./run/channel395/
   ~~~
   {: .bash}

   ~~~
   0  0.orig  Allrun  constant  system
   ~~~
   {: .output}

<p>&nbsp;</p>

## B. Adapt the case to your needs (and Pawsey best practices)

This part does not really makes use of any container.
As the case is already in the host file system, it can be prepared (by editing dictionaries and initial conditions)
as usual.

In this case, the preparation script will make changes to the `$caseDir/system/controlDict` dictionary to comply with Pawsey's best practices for using OpenFOAM.
Best practices have been explained in detail in our [OpenFOAM documentation](https://support.pawsey.org.au/documentation/display/US/OpenFOAM)

   > ## The `B.adaptCase.sh` script (sections to be explained):
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
   > {: .bash}
   > 
   > ~~~
   > ##5.2 Defining the use of collated fileHandler of output results 
   > echo "OptimisationSwitches" >> ./system/controlDict
   > echo "{" >> ./system/controlDict
   > echo "   fileHandler collated;" >> ./system/controlDict
   > echo "}" >> ./system/controlDict
   > ~~~
   > {: .bash}
   {: .solution}

1. Submit the adaptation script

   ~~~
   zeus-1:*-v1912> sbatch --reservation=$myReservation B.adaptCase.sh 
   ~~~
   
   ~~~
   Submitted batch job 4632548
   ~~~
   {: .output}

2. Check the adapted settings in the `controlDict` dictionary

   ~~~
   zeus-1:*-v1912> less ./run/channel395/system/controlDict
   .
   .
   .
   q
   ~~~
   {: .bash}

   > ## Adapted settings in `./run/channel395/system/controlDict` 
   > 
   > - To keep only a few result directories at a time (10 maximum in this case)
   >    ~~~
   >    purgeWrite      10;
   >    ~~~
   > 
   > - To use binary writing format to accelerate writing and reduce the size of the files
   > 
   >    ~~~
   >    writeFormat     binary;
   >    ~~~
   > 
   > - Never use `runTimeModifiable`. This option creates permanent reading of dictionaries (each time step) which overloads the shared file system.
   > 
   >    ~~~
   >    runTimeModifiable false;
   >    ~~~
   > 
   > - If version is higher-or-equal than OpenFOAM-6 or OpenFOAM-v1812, always use the collated option
   > 
   >    ~~~
   >    optimisationSwitches
   >    {
   >        fileHandler collated;
   >    }
   >    ~~~
   {: .solution}

<p>&nbsp;</p>

## C. Decomposition

1. Submit the decomposition script (from the scripts directory)

   ~~~
   zeus-1:*-v1912> sbatch --reservation=$myReservation C.decomposeFoam.sh 
   ~~~
   
   ~~~
   Submitted batch job 4632558
   ~~~
   {: .output}

2. Check that the decomposition has been performed:

   ~~~
   zeus-1:*-v1912> ls ./run/channel395/processor*
   ~~~
   {: .bash}

   ~~~
   ./run/channel395/processors4_0-1:
   0  constant
   
   ./run/channel395/processors4_2-3:
   0  constant
   ~~~
   {: .output}

## D. Using the solver

1. Submit the solver script (from the scripts directory)

   ~~~
   zeus-1:*-v1912> sbatch --reservation=$myReservation D.runFoam.sh 
   ~~~
   
   ~~~
   Submitted batch job 4632685 on cluster zeus
   ~~~
   {: .output}

2. Check that the solver is running:

   ~~~
   zeus-1:*-v1912> squeue -u $USER
   ~~~
   {: .bash}

   ~~~
   JOBID    USER     ACCOUNT     PARTITION            NAME EXEC_HOST ST     REASON   START_TIME     END_TIME  TIME_LEFT NODES   PRIORITY
   4632685  espinosa pawsey0001  workq        D.runFoam.sh       n/a PD  Resources     17:09:28     17:19:28      10:00     1      75190 
   ~~~
   {: .output}

   Observe the output of the job with `tail -f` at runtime (use `<Ctrl-C>` to exit the command):
   ~~~
   zeus-1:*-v1912> tail -f slurm-4632685.out
   ~~~
   {: .bash}

   ~~~
   .
   .
   .
   Time = 0.2
   
   PIMPLE: iteration 1
   smoothSolver:  Solving for Ux, Initial residual = 0.0118746, Final residual = 1.89249e-06, No Iterations 3
   smoothSolver:  Solving for Uy, Initial residual = 0.0617212, Final residual = 1.68113e-06, No Iterations 4
   smoothSolver:  Solving for Uz, Initial residual = 0.0589944, Final residual = 9.70923e-06, No Iterations 3
   Pressure gradient source: uncorrected Ubar = 0.13369, pressure gradient = -0.000964871
   GAMG:  Solving for p, Initial residual = 0.213844, Final residual = 0.00414884, No Iterations 2
   time step continuity errors : sum local = 5.82807e-06, global = -1.41211e-19, cumulative = -1.41211e-19
   Pressure gradient source: uncorrected Ubar = 0.133687, pressure gradient = -0.000947989
   GAMG:  Solving for p, Initial residual = 0.0222643, Final residual = 4.30412e-07, No Iterations 7
   time step continuity errors : sum local = 5.63638e-10, global = -2.40486e-19, cumulative = -3.81697e-19
   Pressure gradient source: uncorrected Ubar = 0.133687, pressure gradient = -0.000947874
   ExecutionTime = 0.25 s  ClockTime = 0 s
   .
   .
   .
   ~~~
   {: .output}

3. Check that the solver gave some results:

   ~~~
   zeus-1:*-v1912> ls ./run/channel395/processor*
   ~~~
   {: .bash}

   ~~~
   ./run/channel395/processors4_0-1:
   0  10  8.2  8.4  8.6  8.8  9  9.2  9.4  9.6  9.8  constant
   
   ./run/channel395/processors4_2-3:
   0  10  8.2  8.4  8.6  8.8  9  9.2  9.4  9.6  9.8  constant
   ~~~
   {: .output}

   > ## The `B.adaptCase.sh` script (sections to be explained):
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
   > {: .bash}
   > 
   > ~~~
   > ##5.2 Defining the use of collated fileHandler of output results 
   > echo "OptimisationSwitches" >> ./system/controlDict
   > echo "{" >> ./system/controlDict
   > echo "   fileHandler collated;" >> ./system/controlDict
   > echo "}" >> ./system/controlDict
   > ~~~
   > {: .bash}
   {: .discussion}

#### 7.C submit the decomposition job:

```bash
$ cd $scriptsDir
$ sbatch --reservation=$myReservation C.decomposeFoam.sh
Submitted batch job 4619406 on cluster zeus
```

Now we can check if the decomposition was performed correctly. You can read the slurm output for the job (use the number of your own `SLURM_JOBID`, 4619406 in this case):

```bash
$ cd $scriptsDir
$ less slurm-4619406.out
.
.
.
q
```

You can also cd to the working directory and browse it:

```bash
$ cd $caseDir
$ ls
0  0.orig  Allrun  constant  logs  processors4_0-1  processors4_2-3  system
$ ls processors4_0-1
0  constant
```
where you can notice the decomposed directories.

And check the specific log file for the decomposition created by the `tee` command in the job script:

```bash
$ cd $caseDir
$ less logs/pre/log.decomposePar.4619406
.
.
.
q
```

<p>&nbsp;</p>

## 8. PAUSE: If you did not reach the correct decomposition, then use this:

If you have not obtained the right decomposition, you can start all over again, but this time execute the extraction and preparation scripts too.

Check the directory where we want the solution to be:

```bash
$ echo $caseDir
/scratch/pawsey0001/espinosa/OpenFOAM/espinosa-v1912/workshop/01_usingOpenFOAMContainers/run/channel395
```

Then remove the directory. (**Always be careful with the rm command as you can't undo it**):

```bash
$ rm -rf $caseDir
```

Now go to the scripts directory and submit your scripts. Wait for them to finish before executing the next one.

The extract and prepare are not submitted with reservation becasue they use the copyq:

```bash
$ cd $scriptsDir
$ sbatch A.extractTutorial.sh 
Submitted batch job 4619469
```

```bash
$ cd $scriptsDir
$ sbatch B.prepareCase.sh 
Submitted batch job 4619471
```
For the decomposition script use the reservation if available:

```bash
$ cd $scriptsDir
$ sbatch --reservation=$myReservation C.decomposeFoam.sh 
Submitted batch job 4619473 on cluster zeus
```

You should have finished with a correctly decomposed case:

```bash
$ cd $caseDir
$ ls
0  0.orig  Allrun  constant  logs  processors4_0-1  processors4_2-3  system
$ ls processors4_0-1
0  constant
```

<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>

## 9. The fileHandler collated option

The default for common OpenFOAM installations is to output results of a decomposed simulation into many processor* subdirectories. Indeed, as many as decomposed subdomains. So, for example, in a common installation you would have finished with the following directories in your caseDir:

```bash
0  0.orig  Allrun  constant  logs  processor0  processor1  processor2  processor3  system
```

So, the default fileHandler for common installations is **uncollated**. That is not a problem for a small case like this tutorial, but it is indeed a problem when the decomposition requires hundreds of subdomains. Then if results are saved very frequently, and the user needs to keep all their output for analysis (purgeWrite=0), then they can end with millions of files. And the problem increases if the user needs to analyse and keep results for multiple cases.

The existence of millions of files is already a problem for the user, but is also a problem for all our user base, because the file system manager gets overloaded and the performance of the file system (which is shared) degrades.

Because of that, for versions greater or equal to OpenFOAM-6 and OpenFOAM-v1812, the use "**fileHandler collated**" is a required policy in our systems. (Older versions do not have that option or do not perform well.) The collated option allows result files to be merged into a reduce number of directories. (We have set that in point 5.B (above) within the controlDict, although for Pawsey containers and system-wide installations, that option is already set by default. We decided to explicitly include it in the controlDict as a reminder to our users.)

Together with the setting of the collated option, the merging of results is controlled by the **ioRanks** option. In this casem the option is being set through the **FOAM_IORANKS** variable. In the scripts we have:

```bash
export FOAM_IORANKS='(0 2 4 6 8)'
```
which indicates that decomposed results will be merged every 2 subdomains. [0-1] in the first subdirectory, and [2-3] in the second one. As seen in our case directory:

```bash
0  0.orig  Allrun  constant  logs  processors4_0-1  processors4_2-3  system
```
(The extra numbers in the list does not affect the settings, but strictly speaking '(0 2)' would be enough.) The "export" is needed for the variable to be exported to the srun executions.

If the ioRanks option were not set, then the collated option would merge all the results into a single directory, as this:

```bash
0  0.orig  Allrun  constant  logs  processors4  system
```

In production runs, we recommend to group results for each node into a single directory. So, for a case to be solved on Magnus, the recommended setting is:

```bash
export FOAM_IORANKS='(0 24 48 72 . . . numberOfSubdomains)'
```
and for a case to be solved on Zeus:

```bash
export FOAM_IORANKS='(0 28 56 84 . . . numberOfSubdomains)'
```
If the numberOfSubdomains fit in a single node, we recommend to use the plain collated option without ioRanks.

Also, we recommend to decompose your domain so that subdomains count with ~100,000 cells. This rule of thumb gives good performance in most of situations, although users need to verify the best settings for their own cases and solvers.

For more information, please read our documentation [OpenFOAM documentation](https://support.pawsey.org.au/documentation/display/US/OpenFOAM).

<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>

## 10. Submit the rest of the jobs (solver and reconstruction)

#### 10.D Execute the solver

Take a look into the script:

```bash
$ cd $scriptsDir
$ less D.runFoam.sh
.
.
.
q
```

Submit the job:

```bash
$ cd $scriptsDir
$ sbatch --reservation=$myReservation D.runFoam.sh
Submitted batch job 4619611
$ squeue -u $USER
JOBID    USER     ACCOUNT     PARTITION            NAME EXEC_HOST ST     REASON   START_TIME     END_TIME  TIME_LEFT NODES   PRIORITY
4619611  espinosa pawsey0001  workq        D.runFoam.sh      z064  R       None     15:19:52     15:29:52       9:50     1      75185
$ tail -f slurm-4619611.out
.
.
.
<ctrl-c>
```

Check if results exist:

```bash
$ cd $caseDir
$ ls
0  0.orig  Allrun  constant  logs  processors4_0-1  processors4_2-3  system
$ ls processors4_0-1/
0  13.2  13.4  13.6  13.8  14  14.2  14.4  14.6  14.8  15  constant
```

#### 10.E Perform reconstruction

Take a look into the script:

```bash
$ cd $scriptsDir
$ less E.reconstructFoam.sh
.
.
.
q
```

Submit the job:

```bash
$ cd $scriptsDir
$ sbatch --reservation=$myReservation E.reconstructFoam.sh
Submitted batch job 4619652 on cluster zeus
```
The reconstructed directory, which should be the latest time existing in the results should now be in the working directory:

```bash
$ cd $caseDir
$ ls
0  0.orig  15  Allrun  constant  logs  processors4_0-1  processors4_2-3  system
```

<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>

## 11. Further notes on how to use OpenFOAM and OpenFOAM containers at Pawsey

The usage of OpenFOAM and OpenFOAM containers at Pawsey has already been described in our documentation: [OpenFOAM documentation at Pawsey](https://support.pawsey.org.au/documentation/display/US/OpenFOAM)

and in a technical newsletter note: [https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04](https://support.pawsey.org.au/documentation/display/US/Pawsey+Technical+Newsletter+2020-04) 
