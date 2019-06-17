---
title: "Docker All the Things Nginx and Supervisor"
date: 2013-12-14T16:21:56-07:00
coverImage: /images/docker.jpg
coverMeta: out
coverSize: partial
thumbnailImagePosition: right
tags:
- docker
- opensuse
categories:
- containerized deployment
---

## WARNING: Much of the content here no long exists or is no longer relevant.  Of most pressing note is that the SUSE Studio service has been merged with the Open Build Service (https://openbuildservice.org/2017/09/27/suse-studio-express/) changing much of the instructions about building the JeOS images referenced here.  I have also deleted the referenced Docker images and Github repositories in this post since they are no longer being maintained.  If interested, I can make an updated version of this post using more modern tools.  For now however, I am leaving this post as an archived reference.

The Beginning
-------------

Recently the DevOps community has been all a hype around the potential of Docker to help change the way that we deploy applications, and its potential to help streamline LXC into a more grokable package just like Vagrant did for virtualized developer environments.  With finals finally over, and the Winter break upon me, I have begun to take the time to rearchitecture some of the server applications that I use everyday: mainly my web hosting, ownCloud, and perhaps mail and database servers if I begin to feel the need.  To do this, I have decided to try out the potential of applying the concepts and capabilities of Docker in order to change the way that I administer my personal applications, and gain some understanding that I can take into my professional life.  Here begins my adventure into using Docker to build my infrastructure, one application at a time, in a way that I believe will make my future administration of these services a much easier task.

Digital Ocean
-------------

I have chosen to use Digital Ocean to host my applications, though the following steps should apply to any server host, Digital Ocean provides a fast and cheap VPS option that you might want to try yourself in order to replicate the steps that I have done.  If you do choose Digital Ocean as your hosting provider, please do use my referral [link](https://www.digitalocean.com/?refcode=60c2cf22747f).

Setting Up The Docker Host
--------------------------

Before we can start building and deploying Docker containers, we need a system with Docker installed to host and manage our running containers.  Though Docker since version 0.7 (I am using 0.7.1 at this moment) should work on almost any Linux distribution, it is still officially supported only on Ubuntu releases, and works best on systems with AUFS support.  Since Ubuntu already has AUFS support packaged and in its kernel, it is the natural base system to choose.    I will not go into the installation process in detail, simply read the Docker guide [here](http://docs.docker.io/en/latest/installation/ubuntulinux/).  Once you have Docker installed and working, continue to the next section.

Building And Running An Nginx Image
-----------------------------------

As I am an openSUSE user on the desktop, and think that it is a very well put together Linux distribution for both server and personal use, I have decided to base all of my Docker containers off of an openSUSE base.  Searching the Docker index for openSUSE images initially brought no results, so I decided to build my own base image and publish it on the index, so that I and others can easily build new application images based on openSUSE.  Using the distribution build tool KIWI, I built a JeOS [image](https://index.docker.io/u/mmckeen/opensuse-13-1/) based on openSUSE 13.1, which also happens to be a long term support Evergreen release, and should provide for a constantly patched base to use for hosting applications.

This JeOS image contains the bare minimum of packages, not even including such programs as wget and curl.  In this way, it is perfect for only running a single application per container, like Nginx or MariaDB, so that each container contains the bare system neccessary to serve a certain purpose, and no extra packages.  Each container provides the smallest attack surface possible for each application being contained, as well as an easy way to bundle all the system dependencies necessary to run one, and only one application, in the same way on any Docker host.

Update (2013-12-17):
As of today, the mmckeen/opensuse-13-1 image of mine on the index has been cut down to a third of its size by removing all of the graphics related packages
that are still part of the JeOS KIWI template.  It should now create much smaller images, and be of better use with Docker. 

In order to build the Nginx image, I used the Dockerfile below:

```
FROM mmckeen/opensuse-13-1  
MAINTAINER Matthew McKeen <matthew@mmckeen.net>

RUN zypper -n in nginx

# tell Nginx to stay foregrounded
RUN echo "daemon off;" >> /etc/nginx/nginx.conf

# run
ENTRYPOINT /usr/sbin/nginx -c /etc/nginx/nginx.conf

# expose HTTP
EXPOSE 80
```

This Dockerfile builds a container based on my openSUSE 13.1 base image with preinstalled Nginx, configures Nginx to disable running as a daemon, and auto starts Nginx when the container is run.  It also exposes port 80, allowing you to forward traffic to and from the Nginx server within from the host system.
The [mmckeen/nginx](https://index.docker.io/u/mmckeen/nginx/) image on the Docker index is auto-built directly from my Github [dockerfiles](https://github.com/mmckeen/dockerfiles) repository, at this time by using the above Dockerfile.  To pull this image for your own use, simply run `docker pull mmckeen/nginx`.

At this point, we are ready to spin up the container and run Nginx with a simple 

`docker run -p 80:80 mmckeen/nginx`  

Using the -p option to forward the container port 80 to the host port 80, simply visiting [http://localhost:80](http://localhost:80) gives us the expected Nginx 403 Forbidden page, since we have not given the server any files to serve.  To do this, we can use the docker run -v option to mount a host local directory to the default Nginx document root inside the container. 

`docker run -v /srv/www/mmckeen.net:/srv/www/htdocs -p 80:80 mmckeen/nginx`

This mounts my website code at /srv/www/mmckeen.net on the Ubuntu host to the container, served by Nginx from openSUSE.  Visiting [http://localhost:80](http://localhost:80) now will serve whatever files you have stored on the host machine from within the isolated environment of the container.

Supervisor And Docker = Awesome
-------------------------------

Now that we have Nginx running in our container, we need some kind of process control to control the running of this container, enable auto respawn of the container if it should exit, and make our Nginx container run in a similar way to to a Upstart, systemd, or standard SysVinit service.  If I was running on a openSUSE, Arch Linux, or Fedora server I would write a systemd unit to control the running of the container.  Because I hate Upstart with a passion, and an initscript doesn't really allow for decent autostart and dependability functions without the addition of Puppet or other services, I decided to use the simple Supervisor service to control the running of my Docker containers.  On Ubuntu, the package installs it's configuration in `/etc/supervisor/supervisord.conf` with user configuration in the `/etc/supervisor/conf.d` directory.  Making the Nginx container autostart is a simple matter of putting the following into a `.conf` file in the `conf.d` directory: in my case in `/etc/supervisor/conf.d/mmckeen.net.conf`.

```
[program:mmckeen.net]
command=/usr/bin/docker run -a stdout -rm --name=mmckeen.net -v /srv/www/mmckeen.net:/srv/www/htdocs -p 80:80 mmckeen/nginx
autorestart=true
```

Notice the addition of several options to our original commands. `-a stdout` is used to direct the container output to stdout so that Supervisor can detect and log the output from the container.  `-rm` deletes the container once it exists, and `--name=mmckeen.net` gives our container a name so that we can not have two containers existing at the same time from the same command, and if any container fails, there will be no name conflicts with the new container being created.  Doing this will automatically restart the container if it fails, as well as make it start at boot.  We are now successfully serving a website via Nginx from within a Docker container.

Security Considerations
-----------------------
Running a website this way does not completely isolate your site from the security considerations of running it directly via Nginx on the host, in fact it adds more security considerations.  The most important of these is that the Docker daemon's API does not require any kind of authentication.  It is important to make sure that you have good firewall in place to isolate the host machine from the outside world, or anyone might be able to control your Docker daemon.  Also take into account that Docker's bridged networking as well as mounted filesystem support allows for possible security holes into and out of the container itself, just as these features do with traditional virtualization.
Of course. the topic of container security is a massive subject, but keep in mind not to keep the security of LXC and Docker for granted.

Conclusion, For Now...
----------------------
This is of course just a simple use of Docker.  In this case, we are serving a static website, and don't have to worry very much about changing application state,
but it introduces the basic concepts that you will need to start using Docker to serve your own applications.  As I continue to build out more services, I will add more blogs posts on my exploits.  Hopefully, you will find them useful as well.
