---
layout: post
title: "Docker debug with nsenter on boot2docker"
date: 2014-07-05 14:05:41 +0200
comments: true
categories: [Docker, DevOps, Boot2docker, nsenter]
author: Lajos Papp
published: true
---

`nsenter` is a small tool allowing to `enter` into `n`ame`s`paces. Specifically
when you work with docker, it means you can *enter* any docker container, even
it they don't run any sshd. Running sshd in a docker container for debuging
[considered evil](http://jpetazzo.github.io/2014/06/23/docker-ssh-considered-evil/).

## Nsenter with Boot2docker

Docker doesn't run directly on OS X and on Windows, so you need
[boot2docker](http://boot2docker.io/). To get `nsenter` working with boot2docker
is a bit trickier.

For the impatient here is a simple function, which lets you enter any docker
container directly from OS X (or any boot2docker host):

```
docker-enter() {
  boot2docker ssh '[ -f /var/lib/boot2docker/nsenter ] || (docker run --rm -v /var/lib/boot2docker/:/target jpetazzo/nsenter ; sudo curl -Lo /var/lib/boot2docker/docker-enter https://raw.githubusercontent.com/jpetazzo/nsenter/master/docker-enter )'
  boot2docker ssh -t sudo /var/lib/boot2docker/docker-enter "$@"
}
```

<!-- more -->

## TL;DR

If you are interested about the details how this works read on.

## Install nseneter onto boot2docker

How to install nsenter into boot2docker? Its a bit tricky, as boot2docker isn't
a full-blown linux, it's based on tiny core linux, so compiling on it is not trivial.

But guess what, **jpetazzo** already created a [dockerized nsenter](https://github.com/jpetazzo/nsenter)
It suggest to install the binary `nsenter` as:

```
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
```

This works with boot2docker ... until you restart it. You should store all
changes on the persistent `/var/lib/boot2docker` directory.

```
docker run --rm -v /var/lib/boot2docker/:/target jpetazzo/nsenter
```

## install the docker-enter script

[docker-enter](https://github.com/jpetazzo/nsenter/blob/master/docker-enter) is
a helper script to do the following 2 steps:

- gets the `PID` of the docker container
- executes `nsenter` passing the correct parameters
 
You can install the `docker-enter` helper script into the same directory as 
nsenter via curl:

```
sudo curl -L \
   -o /var/lib/boot2docker/docker-enter \
   https://raw.githubusercontent.com/jpetazzo/nsenter/master/docker-enter
```

## Nsenter directly from OS X

Some blogs advise you to first ssh into boot2docker, and use nsenter or docker-enter
inside of the virtual env. But if you are executing a single command via ssh, you
can pass the command to the last argument of: `boot2docker ssh <COMMAND>`

## One-liner

So combine all the steps into a single **one-liner** function:

```
docker-enter() { boot2docker ssh -t "[ -f /var/lib/boot2docker/nsenter ] || (docker run --rm -v /var/lib/boot2docker/:/target jpetazzo/nsenter ; sudo curl -Lo /var/lib/boot2docker/docker-enter https://raw.githubusercontent.com/jpetazzo/nsenter/master/docker-enter ) ; sudo /var/lib/boot2docker/docker-enter $@"; }
```

If you want it permanently either copy-paste it into your `~/.profile` or 
`~/.bash_profile`. Or save it into `/usr/local/bin`:

```
curl -Lo /usr/local/bin/docker-enter j.mp/docker-enter && . /usr/local/bin/docker-enter
```

