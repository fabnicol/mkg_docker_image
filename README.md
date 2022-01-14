# Gentoo MKG Docker Images

Docker images are released automatically, signed and verified by third-party workflows. They are based on official Docker Gentoo images, supplemented with extra software using the repository Dockerfile. Within the docker container, the VirtualBox machines will be protected from possible external hazards, and reciprocally the host will be mostly immune from potential hazards affecting the nested machines.     

## The short story

Docker images can be automatically downloaded and used to create [an updated (AMD64) Gentoo ISO installer with MKG](https://github.com/fabnicol/mkg).   
The MKG script will manage the docker image for you and build the Gentoo installer within the container.     
To enable this behaviour, just add option `dockerize` to command line with administrative rights and let it go:

`# ./mkg dockerize gentoo.iso`   

You may add other valid MKG options to this command line, provided that they do not imply graphic display or mounts (like `test_emerge`, `build_virtualbox`, `use_clonezilla_workflow=false` or `plot`).   
Then wait for about a day. The ISO installer will be fetched back from the container upon completion of the Docker job.   
If this does not work, try to fetch back your ISO installer using the standard command line:     
    
`#  docker cp mygentoo:release-master/mkg/gentoo.iso . `   
   
(Replace tag `release-master` with `release-gnome` is you checked out the **gnome** branch rather than 
**master**.)

Alternatively, you can manually download a compressed Docker image from the release section, uncompress it and load it:

`# xz -d *.tar.xz && docker load -i mygentoo-release-[master/gnome].tar`   

then run the Docker image and launch MKG within it manually (see below **Using the Docker image**).     
    
## The long story

What follows is aimed at users who wish to keep control over how containers are created.   
  
### Image availability 

Images are made available:    
    
  + in the [Github Releases section](https://github.com/fabnicol/mkg_docker_image/releases), as automated output of Github Actions workflows,      
  + on [Docker Hub](https://hub.docker.com/repository/docker/fabnicol/mkg_docker_image/builds), as *autobuilds*      
  + or pulled from Docker Hub, using the standard invocation:   
    `$ docker pull fabnicol/mkg_docker_image/branch:[tag]`     
    where `branch:[tag]` is for the image version tag corresponding to a given branch.    
   
  To check the availability of version tags, have a look at the [Docker Hub repository](https://hub.docker.com/repository/docker/fabnicol/mkg_docker_image/tags?page=1&ordering=last_updated). Usually latest images are tagged **master:latest** for Plasma desktops or  **gnome:latest** for Gnome desktops.    

### Prerequisites
  
  You will just need to install VirtualBox kernel modules (and their dependencies) on your host computer, which may or may not imply installing the whole VirtualBox package on the host, depending on your platform and package manager. On Gentoo itself, you will just have to merge:  
   
  `app-emulation/virtualbox-modules app-emulation/docker`   
   
  Installing a full virtualization toolchain on your host will not be necessary, all dependencies, including the nested VirtualBox toolchain, being delegated to the Docker container.    

### Limitations
    
  Some limitations currently apply to MKG within Docker containers:   
   
  + `qemu` and `guestfish`-based options (`share_root`, `shared_dir` and `hot_install`) are not (yet) supported.
  + all options that involve `chroot` are not available, i.e. `use_clonezilla_workflow=false`, `test_emerge` and `build_virtualbox`   
  + graphical interface display is not yet supported. 
    
  See [MKG help](https://github.com/fabnicol/mkg/wiki/3.-Command-line-options) for details.   
  Containers are created using a multi-stage build, from official [Gentoo stage3 AMD64 Docker images](https://github.com/gentoo/gentoo-docker-images). They are fully functional Gentoo distributions augmented with a handful of linux utilities and VirtualBox. For packaging purposes, kernel sources under `/user/src/linux`, and the ebuild database under `/var/db/repos/gentoo` have been removed to keep size down. They can be easily restored using the following command line sequence:   
      
`# emerge --sync`  
`# emerge gentoo-sources`  
`# eselect kernel set 1`  
`# cd /usr/src/linux && make syncconfig && make modules_prepare && cd -`    
     
### Building the Docker image

In what follows, replace `1.0` with the tag of choice. A list of valid tags can be obtained by clicking on the Github [tags button on the mkg_docker_image main repository page](https://github.com/fabnicol/mkg_docker_image/tags).   

#### Building Gentoo official portage and stage3 images first

You will need to update docker to at least version 20.10, enable experimental docker features and add the [`buildx` plugin](https://github.com/docker/buildx) if you do not already have installed it.   

First build fresh official Gentoo portage and stage3 images, following indications given by the [official site](https://github.com/gentoo/gentoo-docker-images): download this repository and within it run:  

`# TARGET=portage ./build.sh`  
`# TARGET=stage3-amd64 ./build.sh`  
   
You created gentoo/stage3:amd64 and gentoo/portage:latest with the above commands. Alternatively, you can pull then from Docker Hub:     
      
`# docker image pull docker.io/gentoo/portage`  
`# docker image pull docker.io/gentoo/stage3:amd64`  
     
#### Then download or clone the present repository

In the source directory, run:
   
> $ sudo docker build -t mygentoo:1.0 .   
   
Or using `buildx` (advised):  
   
> $ docker buildx build -t mygentoo:1.0 .     
    
Adjust with the tag name you want (here mygentoo:1.0).   
Allow some time (possibly several hours) to build, as all is built from source.  
   
#### (Optional) Compress the image

For packaging purposes it is advised to compress the resulting image using the experimental version of Docker.
Add `--squash` after `build`:

> $ sudo docker build --squash -t mygentoo:1.0 .   

This experimental feature will cut down the size of the image by half.    
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
    
### Using the Docker image

#### Running MKG within the container
      
+ Say you just built **docker.io/gentoo/mygentoo:1.0**, and as for the other two base images, firt pull it from cache:    
`#docker image pull docker.io/gentoo/mygentoo:1.0`   
+ Now run the container using:  
  
`# docker run [--privileged [-v /dev/cdrom:/dev/cdrom -v /dev/sr0:/dev/sr0 (...)]] \`   
  `-it --entrypoint bash --device /dev/vboxdrv:/dev/vboxdrv -v /dev/log:/dev/log mygentoo:1.0`   
    
The `--privileged` option is only necessary if you are to create a CloneZilla installer as an ISO image within the container.    
The `-v /dev/cdrom ...` option is only necessary if you wish to automatically start burning your ISO installer to optical disc after completion of the building process. It should be adjusted depending on your hardware and platform configuration; these defaults will work on most GNU/Linux platforms but may have to be changed on other *nix operating systems.     

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
    
#### Running MKG from the host

Alternatively you can run your command line from the host, preferably in daemon mode (-d):    
    
`# docker run -dit [--privileged [-v /dev/cdrom:/dev/cdrom -v /dev/sr0:/dev/sr0 (...)]] \`    
   `--device /dev/vboxdrv:/dev/vboxdrv -v /dev/log:/dev/log mygentoo:1.O [your mkg options]`   
   
A nice way to avoid long command lines is to add to your **~/.bashrc**:
   
`alias mkg="sudo docker run -dit [--privileged [-v /dev/cdrom:/dev/cdrom -v /dev/sr0:/dev/sr0 (...)]] \`     
  `--device /dev/vboxdrv:/dev/vboxdrv -v /dev/log:/dev/log $@"`     
    
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
    


  
