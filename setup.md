---
title: Setup
---
For this workshop we'll be using scripts and exercises from a Git Repository.

## Cloning the exercises from the repository into \scratch

To clone the repository, please do the following:

1. First ssh to zeus providing your credentials

    ~~~
    $ ssh user@zeus.pawsey.org.au
    ~~~
    {: .bash}
    
    ~~~
    Password:
    ############################################################################
    #                    Pawsey Supercomputing Centre                          #
    #        Empowering cutting-edge research for Australia's future           #
    #                                                                          #
    ############################################################################
    zeus-1:~>
    ~~~
    {: .output}

2. Create a directory for the training and clone the repository

    ~~~
    zeus-1:~> mkdir -p $MYSCRATCH/pawseyTraining
    zeus-1:~> cd $MYSCRATCH/pawseyTraining
    zeus-1:pawseyTraining> git clone https://github.com/PawseySC/containers-openfoam-workshop-scripts.git
    ~~~
    {: .bash}
    
    ~~~
    Cloning into 'containers-openfoam-workshop-scripts'...
    remote: Enumerating objects: 30, done.
    remote: Counting objects: 100% (30/30), done.
    remote: Compressing objects: 100% (16/16), done.
    remote: Total 30 (delta 10), reused 30 (delta 10), pack-reused 0
    Unpacking objects: 100% (30/30), done.
    ~~~
    {: .output}
{% include links.md %}
