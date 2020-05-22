
## 3. Interactive (run) and non-interactive (exec) execution

You can always execute the Singularity container interactively:

~~~bash
$ theImage="/group/singularity/pawseyRepository/OpenFOAM/openfoam-v1912-pawsey.sif"
$ singularity run $theImage
Singularity>
~~~

Notice that Singularity mounts the directory from where it was called, and it is accessible from within the container:

~~~bash
Singularity> pwd
/group/pawsey0001/espinosa
~~~

Interactive sessions are great for exploring the installations besides reading the definition files. For example, you can check the OpenFOAM environment variables. Type the following and press \<Tab\>:

~~~
Singularity> $FOAM_
~~~

You will see a list of all the environment variables starting with "FOAM_"

~~~
$FOAM_APP          $FOAM_INST_DIR     $FOAM_RUN          $FOAM_SITE_LIBBIN  $FOAM_USER_APPBIN
$FOAM_APPBIN       $FOAM_JOB_DIR      $FOAM_SETTINGS     $FOAM_SOLVERS      $FOAM_USER_LIBBIN
$FOAM_ETC          $FOAM_LIBBIN       $FOAM_SIGFPE       $FOAM_SRC          $FOAM_UTILITIES
$FOAM_EXT_LIBBIN   $FOAM_MPI          $FOAM_SITE_APPBIN  $FOAM_TUTORIALS    
~~~

And you can check the content of any variable:

~~~
Singularity> echo $FOAM_MPI
mpi-system
Singularity> echo $FOAM_INST_DIR
/opt/OpenFOAM
Singularity> echo $FOAM_TUTORIALS
/opt/OpenFOAM/OpenFOAM-v1912/tutorials
Singularity> echo $WM_THIRD_PARTY_DIR
/opt/OpenFOAM/ThirdParty-v1912
~~~

And, indeed, you can check anything about the installation inside the container:

~~~
Singularity> g++ --version
g++ (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0
Copyright (C) 2017 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

Singularity> mpiexec --version
HYDRA build details:
    Version:                                 3.1.4
    Release Date:                            Fri Feb 20 15:02:56 CST 2015
    CC:                              gcc    
    CXX:                             g++    
    F77:                             gfortran   
    F90:                             gfortran   
    Configure options:                       '--disable-option-checking' '--prefix=/usr' '--enable-fast=all,O3' '--cache-file=/dev/null' '--srcdir=.' 'CC=gcc' 'CFLAGS= -DNDEBUG -DNVALGRIND -O3' 'LDFLAGS= ' 'LIBS=-lpthread ' 'CPPFLAGS= -I/tmp/mpich-build/mpich-3.1.4/src/mpl/include -I/tmp/mpich-build/mpich-3.1.4/src/mpl/include -I/tmp/mpich-build/mpich-3.1.4/src/openpa/src -I/tmp/mpich-build/mpich-3.1.4/src/openpa/src -D_REENTRANT -I/tmp/mpich-build/mpich-3.1.4/src/mpi/romio/include'
    Process Manager:                         pmi
    Launchers available:                     ssh rsh fork slurm ll lsf sge manual persist
    Topology libraries available:            hwloc
    Resource management kernels available:   user slurm ll lsf sge pbs cobalt
    Checkpointing libraries available:       
    Demux engines available:                 poll select
Singularity> 
~~~

To exit, simply type:

~~~bash
Singularity> exit
exit
$
~~~

If you want to make use of a more complex interactive session, this should not be performed in the login node. You should always request for the use of a compute node (in the workq or debugq partitions:)

~~~
$ salloc -p debugq -N 4
salloc -p debugq -n 4
salloc: Granted job allocation 4629017
salloc: Waiting for resource configuration
salloc: Nodes z128 are ready for job
@z128:~>
.
.
.
<use your container interactivelly here>
.
.
.
@z128:~> exit
~~~

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

## 4. The `bash -c` command

The `bash` command creates a new bash kernel for execution. Together with the `-c` option, we can define a command to be executed inside that new bash kernel. This command is very important and useful for the execution of OpenFOAM containers as you will notice along this workshop.

For now, we can try to `echo` the content of the `FOAM_TUTORIALS` variable from the command line (non-interactively). First we exemplify some failed attempts and then the use of `bash -c`.

My first try was (did not worked):

~~~bash
$ theImage="/group/singularity/pawseyRepository/OpenFOAM/openfoam-7-pawsey.sif"
$ singularity exec $theImage echo $FOAM_TUTORIALS


~~~
`echo` is ran within the container. But the command did not work because `$FOAM_TUTORIALS` is being evaluated in the host kernel before passing the arguments to the container, and in the host kernel that variable does not exist:

~~~bash
$ echo $FOAM_TUTORIALS


~~~

My second try was (did not worked):

~~~bash 
$ singularity exec $theImage echo '$FOAM_TUTORIALS'
$FOAM_TUTORIALS
~~~
Did not work because `echo` now understands to display the exact string but the content of the variable.


My third try was using `bash -c` with `""` apostrophes (did not worked):

~~~bash
$ singularity exec $theImage bash -c "echo $FOAM_TUTORIALS"


~~~
Almost there, but it did not work because due to the double-apostrophes. Double apostrophes allow the evaluation of the variables inside the quote. Then, `$FOAM_TUTORIALS` is being evaluated again in the host kernel (where it has no value) before passing the arguments to the container. (`echo` is ran within the new bash kernel within the container.)

Finally, this one works:

~~~bash
$ singularity exec $theImage bash -c 'echo $FOAM_TUTORIALS'
/opt/OpenFOAM/OpenFOAM-7/tutorials
~~~
The exact string 'echo $FOAM_TUTORIALS' is passed as an argument without being evaluated by the host kernel. The argument is then received correctly by the new bash kernel inside the container and, inside that kernel `echo $FOAM_TUTORIALS` is being executed. As the variable exists inside the container, then the value can be displayed.

Note that without `bash -c` things do not work either:

~~~bash
$ singularity exec $theImage 'echo $FOAM_TUTORIALS'
/.singularity.d/actions/exec: line 21: exec: echo $FOAM_TUTORIALS: not found
~~~

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
