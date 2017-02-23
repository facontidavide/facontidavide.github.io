1. Install docker https://docs.docker.com/engine/installation/linux/ubuntu/
2. To run docker without super user (optional) http://askubuntu.com/questions/477551/how-can-i-use-docker-without-sudo
3. Computers with NVidia. Install nvidia Docker (to get HW acceleration) https://github.com/NVIDIA/nvidia-docker/wiki


# Usage
## Creating and starting the container:
The command below will **create** the container from the base image if it doesn't exist and log you in. 

```
mkdir -p $HOME/host_docker
docker pull olympusmons/srcsim
nvidia-docker run -it \
--env="DISPLAY" --env="QT_X11_NO_MITSHM=1" --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
-v /dev/log:/dev/log \
--ulimit rtprio=99 \
-v "$HOME/host_docker:/home/user/host_docker" \
-e LOCAL_USER_ID=`id -u $USER` \
-e LOCAL_GROUP_ID=`id -g $USER` -e LOCAL_GROUP_NAME=`id -gn $USER` \
--name valkyrie  olympusmons/srcsim
```


The commands explained:

```nvidia-docker``` wrapper of docker for hw acceleration

```run -it``` create a new **container** from  an image

```--env="QT_X11_NO_MITSHM=1" --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw"``` forward X to host, from [here](http://wiki.ros.org/docker/Tutorials/GUI)

```-v /dev/log:/dev/log``` Fix for errors starting valkyrie in docker, from [here](https://gitlab.com/nasa-jsc-robotics/valkyrie/issues/18)

```--ulimit rtprio=99``` give the docker the ability to use rt priority for ihmc controller

```--name valkyrie``` the name you give to this container

```olympusmons/srcsim``` the **image** to use as a source for this container (https://hub.docker.com/r/olympusmons/srcsim/)


```-v "$HOME/host_docker:/home/user/host_docker" ``` Create a directory on your computer ~/host_docker, mount it inside the docker user's home. Now you can share files between host and the docker container.

This and the one below are part of [a hack](https://github.com/v-lopez/docker_images) so that inside the docker, 
you are not root, but a user with the same user id and group id as in your host machine.

```-e LOCAL_USER_ID=`id -u $USER` ``` 
```-e LOCAL_GROUP_ID=`id -g $USER` -e LOCAL_GROUP_NAME=`id -gn $USER` ``` 

## Working in a single container
Once you are logged in the container, you can execute terminator twice (the first one will crash) and can work from there.

Once you close the original terminal, the container will **stop**. The container will still exist, you can see it if you run `docker ps -a`. To **restart** the container run `docker start -a valkyrie`.

## Deleting the container
After you are doing using your container, you can run `docker rm valkyrie` to delete it. If you start the container with the `--rm` flag, it will be automatically removed after using it.

## Installing stuff on the docker

I believe we should focus on **not doing permanent changes inside the docker**, because they can not be easily reproduced on your team member's containers, and containers are volatile. 

If you need to install something such as a library, I suggest you **create your own Docker** on top of ours. This way when I update the base docker, you just have to rebuild yours on top of mine.

If you have some changes that would like to make to the base docker (some dependency that everyone needs), just make a merge request on this repository's Dockerfile.

Anyway, if you still need to install something just for testing, the root password is `Docker!`, 
just keep in mind that your container will be deleted at some point because we will update the base image.

## Working in multiple containers
The docker philosophy is 
[one process per container](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#/each-container-should-have-only-one-concern). 
Ideally you'd launch a container with the srcsim and other containers for each application, to do so, you need to give each container a 
name (or remove the --name flag and the name is automatically generated), and add at the end of the nvidia-docker command the command you want to execute in the docker ie:

```
nvidia-docker ALL_ARGUMENTS_ABOVE \
--name qual2_simulation olympusmons/srcsim \
roslaunch srcsim qual2.launch init:=true walk_test:=true
```
-
