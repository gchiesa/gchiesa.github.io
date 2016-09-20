---
layout: post
title: My effective Docker setup on OSX
tags: [docker, osx]
---

this how to is aimed to help to setup docker in OSX to behave in the same way it would in a native linux environment.

One of main goal is to make the volumes work flawlessly, without the need of any changes between your local laptop and a dev/prod environment.

Despite I like vagrant because the possibility to describe your virtual environment through a scripting language, in this how-to I personally find more effective the usage of docker-machine in order to create your virtual machine to run docker.

## Assumptions
- the provider used is virtual box
- there  is no ~/.docker folder in your $HOME path
- your docker-machine will be called ‘default’ and there are no other virtual instances with that name on your provider
- you use bash as shell
- you have brew installed on your machine

## Docker Machine installation
Download and install the docker-toolbox project

[https://www.docker.com/products/docker-toolbox](https://www.docker.com/products/docker-toolbox)

## Create the docker-machine
The virtual machine that will run the docker container must be created (only one). If you are behind a corporate proxy (like me!) you must pass the proxy host to the creation command:

~~~
docker-machine create -d virtualbox \
    --engine-env HTTP_PROXY=http://example.com:8080 \
    --engine-env HTTPS_PROXY=https://example.com:8080 \
    --engine-env NO_PROXY=example2.com \
    default
~~~

if you are not behind proxy just type:

~~~
docker-machine create -d virtual box default
~~~

The proxy is necessary in order to make the docker host able to pull the images from the registry and only for that. When you run containers inside you must specify the proxy through env variables in the way you prefer (env variables or arguments)

## Setup the transparent NFS sharing

Docker-machine takes care to transparently manage the mount of volumes inside the container when they are specified by the user ( -v option). But the standard sharing has 3 drawbacks:

1. The shares are really slow

2. The permission exposed for a volume inside the container are the same as the user that run the container: this means that if inside the container you run a process under a different numeric uid than your uid in your laptop the process can’t write on the share.

3. Any command to change ownership or permission on a volume won’t work.
In order to avoid this you can configure your laptop to export the volumes as NFS exports and also to map the root within the container as root on your laptop (this can obviously be a dangerous configuration but you should know what you run inside your container and, anyway, the container can write only in the specified volume / path)

So, we are going to install docker-machine-nfs

[https://github.com/adlogix/docker-machine-nfs](https://github.com/adlogix/docker-machine-nfs)

with brew:

~~~
brew install docker-machine-nfs
~~~

and we configure the machine ‘default’ to be able to use nfs mounts with root mapping:

~~~
docker-machine-nfs default —nfs-config="-alldirs -maproot=0"
~~~

## Configure your bash
You could always use the docker quick start terminal __(Applications-> Docker -> Docker QuickStart Terminal)__ to use docker commands on your laptop but it’s more useful to integrate the setup in your standard bash shell. Create a file called __.dockerenv.rc__  and put this lines in it:

~~~
# DOCKER-MACHINE
 DOCKER_EXPORTS=$(docker-machine env 2>/dev/null)
 if [ "${DOCKER_EXPORTS%% *}" == 'export' ]; then
     eval ${DOCKER_EXPORTS}
 fi
~~~

save and open your __~/.bashrc__ and add the line:

~~~
# DOCKER-MACHINE
 if [[ -s $HOME/dockerenv.rc ]]; then
     source $HOME/dockerenv.rc
 fi
~~~
