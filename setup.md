---
title: Setup
---
> ## What we'll need?
> - You should be able to connect to Zeus via ssh
> - We'll be using scripts and exercises from a Git Repository.
>
{: .prereq}

> ## Confirm you have access on your laptop to the command line. 
> 
> - Linux/Mac users should require no setup. We recommend the usage of native terminal applications. 
> - Windows users require software for SSH connections prior to attending the workshop:
>    - "MobaXTerm" or "Visual Studio Code" are recommended.
> 
> - Info in our documentation: [https://support.pawsey.org.au/documentation/display/US/Logging+in+with+SSH](https://support.pawsey.org.au/documentation/display/US/Logging+in+with+SSH) 
>
{: .callout}

## Connecting to Zeus and Cloning the exercises

> ## Steps for dealing with connection and cloning
> Please do the following:
> 1. Open a linux terminal and `ssh` to zeus providing your credentials
> 
>     ~~~
>     $ ssh user@zeus.pawsey.org.au
>     ~~~
>     {: .bash}
>     
>     ~~~
>     Password:
>     ############################################################################
>     #                    Pawsey Supercomputing Centre                          #
>     #        Empowering cutting-edge research for Australia's future           #
>     #                                                                          #
>     ############################################################################
>     zeus-1:~>
>     ~~~
>     {: .output}
> 
> 2. Create a directory for the training in `/scratch` and clone the repository
> 
>     ~~~
>     zeus-1:~> mkdir -p $MYSCRATCH/pawseyTraining
>     zeus-1:~> cd $MYSCRATCH/pawseyTraining
>     zeus-1:pawseyTraining> git clone https://github.com/PawseySC/containers-openfoam-workshop-scripts.git
>     ~~~
>     {: .bash}
>     
>     ~~~
>     Cloning into 'containers-openfoam-workshop-scripts'...
>     remote: Enumerating objects: 30, done.
>     remote: Counting objects: 100% (30/30), done.
>     remote: Compressing objects: 100% (16/16), done.
>     remote: Total 30 (delta 10), reused 30 (delta 10), pack-reused 0
>     Unpacking objects: 100% (30/30), done.
>     ~~~
>     {: .output}
> 
>     ~~~
>     zeus-1:pawseyTraining> cd containers-openfoam-workshop-scripts/
>     zeus-1:*-scripts> ls
>     ~~~
>     {: .bash}
>     
>     ~~~
>     02_executingFullWorkflow           04_buildingAnOpenFOAMContainer           06_advancedUseOfOverlayFS
>     03_compileAndExecuteUsersOwnTools  05_useOverlayFSForReducingNumberOfFiles  README.md
>     ~~~
>     {: .output}
>     
> 3. To shrink the prompt information (instead of displaying the user and the full current path) copy/paste the following variable set: 
>
>     ~~~
>     espinosa@zeus-1:/scratch/pawsey0001/espinosa/pawseyTraining/containers-openfoam-workshop-scripts> PS1='\[$(ppwd)\]\h:${PWD/*\//}> '
>     ~~~
>     {: .bash}
>
>     ~~~
>     zeus-1:containers-openfoam-workshop-scripts>
>     ~~~
>     {: .output}
>
{: .discussion}


{% include links.md %}
