---
layout: post
title: Dockerizing Jekyll on OS X
date: Sat Aug 23 23:50:23 +0200 2014
timestamp: 19009968
---

_This site is now being served from within a Docker container. Here's what I did to make it happen._

As I've mentioned before, this site is [hosted from home](/posts/indiehost) and [generated using Jekyll](/posts/epoch). The machine it's being hosted on is a Mac Mini, and the actual jekyll process is [managed by a LaunchDaemon](https://github.com/cbeams/chris.beams.io/commit/5f614e68a63694b2c846f4c788dc4b8499aa196c), mainly to ensure that everything comes back up cleanly after a reboot.

This arrangement works pretty well, but I wanted to port it to Docker for several reasons:

1. It's fragile, in that it depends on specific facilities of the underlying OS. And LaunchDaemons suck to work with.
2. I actually plan to move this server away from OS X and to GNU/Linux anyway. This will be easier if the site is already containerized.
3. For reasons explained in my [last post](/posts/docker), I want to experiment with a few ideas around peer-to-peer container hosting. This site is a perfect testing ground.

The move to Docker was more of a hassle than I thought it would be. It took the better part of two days, all told. What follows are the steps I took. I'm still new to Docker, so I'm sure there are better ways to do certain things. Suggestions welcome.


## Install boot2docker

Docker is based on Linux containers, which means that running it on any OS other than Linux is going to require a workaround.

On OS X, [boot2docker](http://boot2docker.io/) is that workaround. It installs VirtualBox and everything else you need to run a Linux virtual machine that will act as your Docker daemon. boot2docker now has a pretty polished installation experience (much appreciated), but the presence of the virtual machine just cannot be made transparent. This isn't boot2docker's fault, it's just a fact of any LXC-based solution: if you're using Docker, you should really be hosting it on Linux. Otherwise things are going to be unnecessarily painful. You've been warned.

Incidentally, [boot2docker 1.2.0](https://github.com/boot2docker/boot2docker/releases/tag/v1.2.0) was released just a few hours prior to my writing this post. I can confirm that it works as advertised. Follow the instructions at <http://docs.docker.com/installation/mac/>.


## Create an image

The next step is to create a Docker image containing everything necessary to run this site. If you're unfamiliar, check out the [relevant section](http://docs.docker.com/userguide/dockerimages/) of the Docker user guide.

To do this, I put together [the following _Dockerfile_](https://github.com/cbeams/chris.beams.io/blob/master/Dockerfile):

```sh
FROM debian:latest
MAINTAINER Chris Beams <chris@beams.io>

RUN apt-get update

# required to build certain native gem extensions depended on by
# jekyll
RUN apt-get install -y make

# required to build 'therubyracer' native gem extension
RUN apt-get install -y g++

# required to clone the site and keep it up to date
RUN apt-get install -y git

# required to listen on $HOOKPORT for update notifications requests
# from GitHub
RUN apt-get install -y netcat

# The -dev version of Ruby 1.9.1 is necessary to avoid
# "cannot load such file -- mkmf (LoadError)" error when installing
# Jekyll.
# See http://askubuntu.com/questions/305884/how-to-install-jekyll
RUN apt-get install -y ruby1.9.1-dev

# Python 3 is installed by default, but this creates a conflict with
# the Pygments highlighting library. The solution is to install
# Python 2.7 and ensure it is available on the $PATH as `python2`.
RUN apt-get install -y python2.7
RUN ln -s /usr/bin/python2.7 /usr/bin/python2

# install the latest jekyll version (2.3.0 at time of writing)
RUN gem install jekyll

# required to avoid ExecJS::RuntimeUnavailable errors when running
# jekyll. See https://github.com/jekyll/jekyll/issues/2327
RUN gem install execjs therubyracer

# clone the site itself
RUN git clone https://github.com/cbeams/chris.beams.io.git
```

As you can see, the file is full of special instructions and tweaks necessary to get Jekyll to work properly. Getting all this right was the largest time sink.

I developed the Dockerfile iteratively by making a change and running the following commands:

    $ docker build --tag="cbeams/chris.beams.io:v2" .
    $ docker run -p 4000:4000 -p 12345:12345 -d cbeams/chris.beams.io:v2 chris.beams.io/serve.sh

As errors cropped up, I would make a change to the Dockerfile, rebuild and re-run. For convenience, I did all of this locally on my laptop as opposed to doing it on the server.


## Share the image

Once I got the image perfected, I pushed it to [a repository](https://registry.hub.docker.com/u/cbeams/chris.beams.io/) under my account at [Docker Hub](https://hub.docker.com/). Docker Hub feels a bit like GitHub, but hosting images rather than code. [Read up on it](http://docs.docker.com/userguide/dockerrepos/) in the user guide if it's new to you.

    $ docker push cbeams/chris.beams.io

My image ended up being around 250 MB, so uploading it from my modest connection at home was a rather slow process. Fortunately, I didn't need to do so very frequently.


## Run the image

Now over on the server (which had also already been outfitted with boot2docker), I ran the following command to fire up the container:

    $ docker run -p 4000:4000 -p 12345:12345 -d cbeams/chris.beams.io:v2 chris.beams.io/serve.sh

Let's break that down:

 - The `cbeams/chris.beams.io:v2` parameter tells the Docker daemon to run the image named `cbeams/chris.beams.io` at tag `v2`—the same image we pushed in the last step. Since the server had not before seen this image, it was automatically downloaded from Docker Hub and then cached locally for future use.

 - The `-d` flag indicates the container should be run in the background as opposed to interactively.

 - The `-p` flags configure which ports are exposed on the container, and which ports on the host system should forward to them. This will be explained in more detail in the step below.

 - The `chris.beams.io/serve.sh` parameter is the [command to be run](https://github.com/cbeams/chris.beams.io/blob/50c89b3a124856aa5c45de8a44f03021ec06a948/serve.sh) within the container. This fires up Jekyll and listens for webhook requests from GitHub.


## Forward all the ports

This is where things get tricky. There are actually _three layers_ of port forwarding in this arrangement:

**1. From cablemodem to OS X server.** This has nothing to do with Docker, of course. Just a one time configuration mapping port 80 on the router to port 4000 on the server.

**2. From OS X server to virtual Linux guest.** The Linux guest in question is _not_ the Docker container. It is the Docker _daemon_ running in VirtualBox as set up by boot2docker. This configuration happens within VirtualBox, and can be done one of two ways:

At the command line with `VBoxManager`:

    $ VBoxManage modifyvm "boot2docker-vm" --natpf1 "jekyll,tcp,,4000,,4000"

Or via the GUI:

![VirtualBox Port Forwarding](http://i.imgur.com/5fgzCiw.jpg)

**3. From virtual Linux guest to Docker container.** Finally, port 4000 must be explicitly exposed on the container itself, and requests to the boot2docker daemon must be forwarded to it. This is configured with the `-p 4000:4000` flag as seen in the step above.

Altogether then, the request flow is:

    router:80 -> server:4000 -> virtualbox:4000 -> container:4000

This is complicated, but it does work (as evidenced by your reading this right now). Keep in mind that the worst part is the VirtualBox configuration, and that goes away entirely if the physical server is a Linux box to begin with.


## Run everything on boot

If the server crashes or gets rebooted, the boot2docker vm should be started automatically and the container for the site should be run.

This is also a bit tricky to get right, and for the moment, I'm only halfway there.

I've added `/Applications/boot2docker` to the login items for the main user on the mac. boot2docker is actually a proper Mac application bundle, so this works well enough. That handles starting the Docker daemon VM, but then one must actually run the container for the site. For now, I'm doing this manually, but I'd like to hear suggestions about better ways to do it. I'm sure there's some way to get it all done using LaunchDaemons, but I really didn't want to waste the time. Andrew Mussey has some prior art in [a tool he calls docker-osx](http://blog.amussey.com/post/85117547548/docker-starting-docker-on-system-boot-on-osx-via), but it's a little outdated at this point (pre-Docker 1.0), so I didn't mess with it.


## Next steps

Now that the basic containerization is complete, I'll take a look at ways to monitor resource consumption at the container level. If this is possible, then it may make sense to take the next step and look into micropayment-based billing for use of those resources. See my [previous post](/posts/docker) for further details about this larger idea.