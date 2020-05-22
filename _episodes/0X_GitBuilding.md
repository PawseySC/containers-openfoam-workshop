
We will explain on detail the building of the containers later in the workshop, but we can take a look now on the definition files we used to create them. The Git Repository is: [https://github.com/PawseySC/pawsey-containers](https://github.com/PawseySC/pawsey-containers).

For example, take a look to the openfoam-v1912 definition files: [https://github.com/PawseySC/pawsey-containers/tree/master/OpenFOAM/basicInstallations/openfoam-v1912](https://github.com/PawseySC/pawsey-containers/tree/master/OpenFOAM/basicInstallations/openfoam-v1912).

In our approach, the first creation step is to build a container with Docker. The Dockerfile in the directory 01_Docker shows (among a lot of other stuff) the following:

~~~docker
# 0. Initial main definintion
# Defining the base container to build from
# In this case: mpich 3.1.4 and ubuntu 18.04; MPICH is needed for crays

FROM pawsey/mpich-base:3.1.4_ubuntu18.04
~~~
So, OpenFOAM containers start from an already existing image with MPICH. The rest of the Dockerfile has the recipe for compiling that specific version of OpenFOAM.

After the Docker container has been built, tested and pushed into Docker Hub. The final step is to port it into Singularity. For that, a small (but very important) definition file is used ("02_PortingToSingularity/Singularity.def"):

~~~Singularity
Bootstrap: docker
From: pawsey/openfoam:v1912

%post
/bin/mv /bin/sh /bin/sh.original
/bin/ln -s /bin/bash /bin/sh
echo ". /opt/OpenFOAM/OpenFOAM-v1912/etc/bashrc" >> $SINGULARITY_ENVIRONMENT
~~~

The last line in the "%post" section allows the OpenFOAM environment definition file to be sourced by default whenever the Singularity container is used. So, as we saw in the section 1 above, we can execute "icoFoam" or any other OpenFOAM tool without sourcing the bashrc.
