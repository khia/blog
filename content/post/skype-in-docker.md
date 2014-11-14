---
Description: How to run multiple instances of skype on your linux desktop
Keywords:
- skype
- docker
Section: post
Slug: skype-in-docker
Tags:
- skype
- docker
- security
Title: Skype in docker
Topics:
- Security
- Docker
date: 2014-11-13
---


Skype in docker
===============

Introduction
------------

If you want to separate your personal life from work you would end up with two Skype accounts.
However there is a problem. You cannot run multiple instances of Skype on a single computer (as single user).
Here comes an idea to run Skype in the docker.
Running Skype in docker is also useful for privacy sensitive individuals like myself.
Since the Skype application is closed source and sends data in encrypted form.
There is no way to know what it is doing and what it is sending out.
By using docker you can prevent access of Skype application to your files on disk.

Previous Attempts
-----------------

There are multiple attempts to dockerize `skype` over the Internet. However most of them install xorg into the container. They also use ssh X11 forwarding to access skype client from the host computer. Can we do better? Let's find out.

Solution
--------

Docker supports passing unix socket from host to the container. If we pass xorg socket our app (in this case Skype but could be anything) we should be able to use it.

This is how you do it:

	docker run -t -i -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=unix$DISPLAY <conatiner> <app>


Hmm Skype also needs sound. This piece took me quite a lot. I read tons of articles and posts on the internet. Until one day I didn't come across this blog post [Running Google chrome in a container](https://www.stgraber.org/category/planet-canonical/). Here is what the author did:

	#!/bin/sh
	PULSE_PATH=$LXC_ROOTFS_PATH/home/ubuntu/.pulse_socket

	if [ ! -e "$PULSE_PATH" ] || [ -z "$(lsof -n $PULSE_PATH 2>&1)" ]; then
		pactl load-module module-native-protocol-unix auth-anonymous=1 \
			socket=$PULSE_PATH
	fi

This is amazing. Even if this technique was used for lxc it must work in the docker. So I refactor it a little bit and here is how it looks.

	cat ~/bin/enable_sound
    PULSE_DIR=$1/.pulse/
    mkdir -p ${PULSE_DIR}
    echo "disable-shm=yes" > ${PULSE_DIR}/client.conf
    PULSE_PATH=${PULSE_DIR}/.socket
    if [ ! -e "$PULSE_PATH" ] || [ -z "$(lsof -n $PULSE_PATH 2>&1)" ]; then
        pactl load-module module-native-protocol-unix auth-anonymous=1 socket=${PULSE_PATH} > /dev/null
    fi
    echo "-e PULSE_SERVER=/data/.pulse/.socket -e QT_X11_NO_MITSHM=1 -v ${PULSE_DIR}/:/data/.pulse"
This script can be used as follows:

    cat ~/bin/skype.run
    #!/bin/bash
    SCRIPT_DIR="$( cd "$( dirname "$0" )" && pwd )"
    if [ "$#" -ne 1 ]; then
      echo "Usage $0 <profile directory>"
      exit 1
    fi
    sound_args=$(${SCRIPT_DIR}/enable_sound $1)
    docker run ${sound_args} <conatiner> <app>

How about camera? Could we pass web camera device to the docker? Apparently we can.

	cat ~/bin/enable_camera
    #!/bin/bash
    if [ -z ${VIDEO_DEV} ]; then
      for device in /dev/video*
      do
        VIDEO_ARGS="--device=${device} -v ${device}:${device} ${VIDEO_ARGS}"
      done
    else
      VIDEO_ARGS="--device=${VIDEO_DEV} -v ${VIDEO_DEV}:${VIDEO_DEV}"
    fi
    echo ${VIDEO_ARGS}

We need to update `~/bin/skype.run` as follows

    - sound_args=$(${SCRIPT_DIR}/enable_sound $1)
    - docker run ${sound_args} <conatiner> <app>
    + sound_args=$(${SCRIPT_DIR}/enable_sound $1)
	+ camera_args=$(${SCRIPT_DIR}/enable_camera $1)
    + docker run ${sound_args} ${camera_arg} <conatiner> <app>

However my camera didn't work with Skype. Skype needs to access some cameras in compatibility mode. In order to use it add following into `Dockerfile`

    # Enable camera
    RUN apt-get install -y libv4l-0:i386
    CMD "LD_PRELOAD=/usr/lib/i386-linux-gnu/libv4l/v4l1compat.so /usr/bin/skype"

Having `enable_video` and `enable_camera` as independent scripts make them very easy to maintain. So I ended up refactoring out xorg socket passing code into `bin/enable_video`. You can see the result in my other dockerized application [docker-chromium](https://github.com/khia-docker/docker-chromium)

Full source code of docker-skype is [available](https://github.com/khia-docker/docker-skype)

Security Notes
--------------

This way of running Skype in docker protects only access to your files. Skype app still would be able to keylog or taking screenshots if it does so. Since it has access to xorg socket.

Ways to improve
---------------

I don't like that we need to deploy 4 scripts to host machine

  - skype.run
  - enable_sound
  - enable_video
  - enable_camera

So I'm looking for ways how to keep them separate in source code but merge into one script on installation. I want the following to produce one single file `~/bin/skype.run`.

	docker run <container> install > ~/bin/skype.run && chmod +x ~/bin/skype.run
