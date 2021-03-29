# Gentoo MKG Docker Images

## Building the container

The container is created using a multi-stage build, which requires Docker >= 19.03.0.
It draws upon the official [Gentoo stage3 AMD64 Docker image](https://github.com/gentoo/gentoo-docker-images)

## Building the Docker image

Download or clone the repository.
In the source directory, run:
   
> $ sudo docker build -t [your tagged image name] .   
   
Allow for some time (possibly several hours) to build, as all is built from source.  

## Using the Docker image in a Dockerfile

Say you just built **mygentoo** with the above command and Dockerfile.   
This image can be reused in multi-stage builds as follows. 
The following Dockerfile updates the image:

    # name the portage image
    FROM mygentoo as build
    
    # Use a current base stage3 image
    FROM gentoo/stage3-amd64:latest
    
    WORKDIR /
    
    # copy the entire root
    COPY --from=build / .
    
    # continue with image build ...
    RUN emerge -auDN --with-bdeps=y @world
    

