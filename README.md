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

In what follows, replace `1.0` with the tag of choice. A list of valid tags can be obtained by clicking on the Github **tags** button on this page.     
In the source directory, run:
   
> $ sudo docker build -t mygentoo:1.0 .   
   
Adjust with the tag name you want (here mygentoo:1.0).   
Allow some time (possibly several hours) to build, as all is built from source.  

### (Optional) Compress the image

For packaging purposes it is advised to compress the resulting image using [docker-squash](https://github.com/jwilder/docker-squash).  
Optionally clean the container of kernel sources:   
   
`# docker run --entrypoint bash -it mygentoo:1.0`       
   
[Note the container ID on return.]   
   
`(container)# rm -rf /usr/src/linux && exit`   
   
Then commit the container and tag it:   
   
`# docker commit ID`   
`# docker tag ID mygentoo:1.1`    
   
Then use docker-squash:   
   
`# docker save ID | docker-squash | docker load`     
   
Finally use **zip** or **xz** compression to archive the squash tarball.     
The resulting compressed tarball is about 15 % the size of the Docker image created by the above build stage.  
    
## Using the Docker image

### Running MKG within the container
      
+ Say you just built **docker.io/gentoo/mygentoo:1.0**, and as for the other two base images, firt pull it from cache:    
`#docker image pull docker.io/gentoo/mygentoo:1.0`   
+ Now run the container using:  
`# docker run -it --entrypoint bash --device /dev/vboxdrv:/dev/vboxdrv -v /dev/log:/dev/log mygentoo:1.0`   
+ Once in the container, note its ID on the left of the shell  input line.   
+ If the image contains an **mkg** directory, run `git pull` within it to update the sources.   
Otherwise (depending on versions), clone the *mkg* repository:   
`# git clone --depth=1 https://github.com/fabnicol/mkg.git`   
and then `cd` to directory **mkg**.   
+ Run your `./mkg` command line, remembering to use `gui=false` and not to use `share_root`, `hot_install`, `from_device`, `use_clonezilla_workflow=false` or `test_emerge`       
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

`alias mkg="sudo docker run -dit --device /dev/vboxdrv:/dev/vboxdrv -v /dev/log:/dev/log \"$@\""`   
  
so that after running `source ~/.bashrc`, you just have to call mkg as if it were an installed script:

`# mkg [your image name first: here mygentoo:1.0] [your mkg argument names: gentoo2.iso ncpus=2 verbose [...]]`    
  
[note the ID when the function returns]  

Note that `gui=false` is already set in this launch mode, so it does not need to be specified (and should not be overridden).  

You can check the container state by shelling back into it:

`# docker exec -it ID bash`    

and within it examine **nohup.out** which logs the job. Then exit as usually (`Ctrl-P, Ctrl-Q`).   

### Switching from Plasma to Gnome and back

You should create one image for Gnome and another for Plasma.   
Just checkout the **gnome** branch of this repository, then build your image and run as above without modification to obtain a Gnome desktop rather than a default Plasma desktop. If you want to preserve both options, it is advised to tag your images accordingly. For example, checkout the **gnome** branch and run:

`# docker build -t mygentoo:gnome-1.0 .`  

When completed checkout back to the **master** branch and run: 

`# docker build -t mygentoo:plasma-1.0 .`  

Then you can run either image using the same `mkg()` function in ~/.bashrc as above.  
   
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
    
### Limitations

Some MKG options do not work within containers. See [MKG help](https://github.com/fabnicol/mkg/wiki/3.-Command-line-options) for details.     
  
