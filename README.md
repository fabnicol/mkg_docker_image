# Gentoo MKG Docker Images

The container is created using a multi-stage build, which requires Docker >= 19.03.0.
It draws upon the official [Gentoo stage3 AMD64 Docker image](https://github.com/gentoo/gentoo-docker-images)

## Building the Docker image

### Building Gentoo official portage and stage3 images first

You will need to update docker to at least version 20.10, enable experimental docker features and add the [`buildx` plugin](https://github.com/docker/buildx) if you do not already have installed it.   

First build fresh official Gentoo portage and stage3 images, following indications given by the [official site](https://github.com/gentoo/gentoo-docker-images): download this repository and within it run:  

`# TARGET=portage ./build.sh`  
`# TARGET=stage3-amd64 ./build.sh`  

You created **docker.io/gentoo/stage3:amd64** and **docker.io/library/newgentoo:1.0** with the above commands. You will have to pull then from cache before using them:

`#docker image pull docker.io/gentoo/portage`    
`#docker image pull docker.io/gentoo/stage3:amd64`   
   
### Then download or clone the present repository.

In the source directory, run:
   
> $ sudo docker build -t mygentoo:1.0 .   
   
Adjust with the tag name you want (here mygentoo:1.0).   
Allow for some time (possibly several hours) to build, as all is built from source.  

## Using the Docker image in a Dockerfile

### Running MKG within the container
      
+ Say you just built **docker.io/gentoo/mygentoo:1.0**, and as for the other two base images, firt pull it from cache:    
`#docker image pull docker.io/gentoo/mygentoo:1.0`   
+ Now run the container using:  
`# docker run -it --device /dev/vboxdrv:/dev/vboxdrv -v /dev/log:/dev/log mygentoo:1.0 bash`   
+ Once in the container, note its ID on the left of the shell  input line.   
+ If the image contains an **mkg** directory, run `git pull` within it to update the sources.   
Otherwise (depending on versions), clone the *mkg* repository:   
`# git clone --depth=1 https://github.com/fabnicol/mkg.git`   
and then `cd` to directory **mkg**.   
+ Run your `./mkg` command line, remembering to use `gui=false` and not to use `share_root`, `shared_dir` or `hot_install`      
+ Preferably use:     
`# nohup (...) &`    
so that you can monitor the build in **nohup.out**   
+ Once the virtual machine is safely launched, monitor the run using:    
`# tail -f nohup.out`     
+ Once the process is safely running, exit using Ctrl - p Ctrl - q.    
+ You may come back again into the container by running:        
`#  docker exec -it ID bash`     
+ You may follow the build from your host by running:   
`# docker cp ID:/mkg/nohup.out . && tail -n200 nohup.out `     
  
### Running MKG from the host

Alternatively you can run your command line from the host, preferably in daemon mode (-d):    

`# docker run -dit --device /dev/vboxdrv:/dev/vboxdrv -v /dev/log:/dev/log mygentoo:1.O [your mkg options]`  

A nice way to avoid long command lines is to add to your **~/.bashrc**:

`mkg() { docker run -dit --device /dev/vboxdrv:/dev/vboxdrv -v /dev/log:/dev/log mygentoo:1.O "$@" }`  

so that after running `source ~/.bashrc`, you just have to call mkg as if it were an installed script:

`mkg gentoo2.iso ncpus=2 verbose [...]`    
  
[note the ID when the function returns]  

You can check the container state by shelling back into it:

`docker exec -it ID bash`

and within it examine **nohup.out** which logs the job. Then exit as usually (`Ctrl-P, Ctrl-Q`).   


### Reusing MKG Docker images 

Images built as indicated above or released in the Release section can be reused in multi-stage builds as follows.      
The following Dockerfile updates the image:     

    # name the portage image
    FROM mygentoo:1.0 as build
    
    # Use a current base stage3 image
    FROM gentoo/stage3:amd64
    
    WORKDIR /
    
    # copy the entire root
    COPY --from=build / .
    
    # continue with image build ...
    RUN emerge -auDN --with-bdeps=y @world
    

