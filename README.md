# Gentoo MKG Docker Images

The container is created using a multi-stage build, which requires Docker >= 19.03.0.
It draws upon the official [Gentoo stage3 AMD64 Docker image](https://github.com/gentoo/gentoo-docker-images)

## Building the Docker image

### Building Gentoo official portage and stage3 images first

You will need to update docker to at least version 20.10, enable experimental docker features and add the [`buildx` plugin](https://github.com/docker/buildx) if you do not already have installed it.   

First build fresh official Gentoo portage and stage3 images, following indications given by the [official site](https://github.com/gentoo/gentoo-docker-images): downlad this rpository and within it run:  

`# TARGET=portage ./build.sh`  
`# TARGET=stage3-amd64 ./build.sh`  

You created **docker.io/gentoo/stage3:amd64** and **docker.io/library/newgentoo:1.0** with the above commands. You will have to pull then from cache befor using them:

`#docker image pull docker.io/gentoo/portage`    
`#docker image pull docker.io/gentoo/stage3:amd64`   
   
### Then download or clone the present repository.

In the source directory, run:
   
> $ sudo docker build -t mygentoo:1.0 .   
   
Adjust with the tag name you want (here mygentoo:1.0).   
Allow for some time (possibly several hours) to build, as all is built from source.  

## Using the Docker image in a Dockerfile

Say you just built **docker.io/gentoo/mygentoo:1.0**, and as for the other two base images, firt pull it from cache:

`#docker image pull docker.io/gentoo/mygentoo:1.0`   
  
This image can be reused in multi-stage builds as follows. 
The following Dockerfile updates the image:

    # name the portage image
    FROM mygentoo:1.0 as build
    
    # Use a current base stage3 image
    FROM stage3:amd64
    
    WORKDIR /
    
    # copy the entire root
    COPY --from=build / .
    
    # continue with image build ...
    RUN emerge -auDN --with-bdeps=y @world
    

