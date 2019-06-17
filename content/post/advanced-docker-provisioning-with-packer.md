---
title: "Advanced Docker Provisioning With Packer"
date: 2013-12-27T16:44:37-07:00
coverImage: /images/packer.png
coverMeta: out
coverSize: partial
thumbnailImagePosition: right
tags:
- docker
- packer
- vagrant
categories:
- containerized deployment
---

## WARNING: The openSUSE base image used here is no longer available due to SUSE Studio being deprecated in favor of the Open Build Service (https://openbuildservice.org/2017/09/27/suse-studio-express/).  Much of the content here should still be decently accurate however, so I am keeping this post as an archive and reference.

When Dockerfiles Aren't Enough
-------------------------------

Dockerfiles are a limited solution to a complex problem: the provisioning of Docker images.  Dockerfiles at their core operate much like a simple shell script, being a list of instructions to get from a certain state, or base image, to that of a final state, or output image.  These instructions do not easily change, cannot take on branching logic, and at the same time are just plain ugly to look at.  With all the advancement that modern configuration management systems provide, Dockerfiles are a huge roadblock when considering Docker as a serious contender for use in the repeatable deployment and configuration of complex application stacks.

Even with Dockerfiles, it is possible to hack around their limitations by installing Puppet or Chef, copying modules or cookbooks into the running container, and doing configuration via existing well proven methods of configuration management, but even then we still rely on the Dockerfile language itself to do much of the provisioning work for us.  The work that one does to get Dockerfile provisioning to work cannot translate over to building out virtual and bare metal machines, making it harder to use the same Puppet or Chef code base to configure all of the various types of machines that one may rely on between development, production, and testing.  That is, until we introduce the completely awesome tool that is [Packer](http://packer.io).

Packer
------

Packer is a tool from [Mitchell Hashimoto](http://about.me/mitchellh), the same 
guy who brought us [Vagrant](http://vagrantup.com), designed to allow the same 
provisioners, whether that be Puppet, Chef, or plain shell scripts, to spit out 
anything from images for Digital Ocean or EC2 to Vagrant boxes, and, you've 
guessed it, Docker images.

The benefit of this is obviously that with a small amount of additional configuration in our Packer templates, and with the same provisioning code, I can create an application image that can run anywhere from Vagrant for testing and development, to scaling out on EC2 to thousands of users.  It is truly an awesome automation tool.

Packer and Docker Together At Last
----------------------------------

Using Packer, we can improve how we configure and deploy code to our Nginx Docker image from my previous blog [post](http://mmckeen.net/blog/2013/12/14/docker-all-the-things-nginx-and-supervisor/).  In this post, we allowed the Nginx container access to the files it must serve via a shared folder between the container and the host system.  In a perfect world, the image that we run as a container will encompass all of the dependencies necessary to allow that application to run.  This of course should include the application code being served, and as an added bonus of packaging the code with the image, each image iteration becomes a release of the software being served.  Rollbacks become dead simple, since if we want to run an old version of the code base we simply replace the currently running container with another based on an older image.  The code is contained within the image itself, so whenever we export that image, or host it up on a Docker index, the code goes along with it.  Though moving the code into the image itself could have easily been done with a Dockerfile, I am going to show you how using Packer, I can take the same application and deploy it two ways: in a Vagrant box and in a Docker container with the ability for more deployment platforms only a couple of lines of code away.

We will start with the Docker container, and you'll be amazed at how easy it is.

This is the Packer template for a Docker image that will run this very website that you are now browsing:

```
{
  "builders": [
    {
      "type": "docker",
      "image": "mmckeen/opensuse-13-1",
      "export_path": "mmckeen.net.tar"
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "inline": [
                  "zypper -n in nginx",
                  "echo 'daemon off;' >> /etc/nginx/nginx.conf"
                ]
    },
    {
      "type": "file",
      "source": "/srv/www/mmckeen.net/",
      "destination": "/srv/www/htdocs/"      
    }
  ]
}
```
A Packer template is quite a simple JSON document with this particular document having two sections: builders and provisioners.  The builders section describes the output of the Packer build, and the provisioners provide the code necessary for the build itself.  In the template above the simplest provisioner, which simply runs a set of shell commands, is used in very much the same way as a Dockerfile's `RUN` commands, but the provisioners section also has the ability to use Puppet, Chef, and various other sources to do its provisioning, allowing for much more advanced provisioning options than simple shell commands or scripts.  Also notice that the above template uses the file provisioner to copy the website code base into the final build artifact, allowing for the easy deployment of our code directly into the built Docker image.

And so you can see how this template could be used in action, below is a sample deployment script that handles the building of a new website image, as well as the replacement of the currently running Docker container with the new image.

```
#!/bin/bash

# Do a Packer build

rm -f mmckeen.net.tar

packer build ./packer-build-templates/docker/mmckeen.net/mmckeen.net.json

# Get the latest base image ID
LAST_IMPORT=`cat last_mmckeen_net`

# Import build artifact as Docker image
echo "Importing the newly built image into docker..."
docker import - mmckeen_net_packer < mmckeen.net.tar > last_mmckeen_net

# Delete the previous base image
echo "Removing the old base image..."
docker rmi $LAST_IMPORT

# Build a new production Docker image using metadata only Dockerfile
echo "Add in Docker metadata"
docker build -rm -t="mmckeen_net_prod"  dockerfiles/mmckeen.net/
echo "Final mmckeen_net_prod image built, ready for docker run"

echo "Restarting mmckeen.net through supervisorctl"
supervisorctl restart mmckeen.net
```

The script is fairly simple, doing the Packer build, importing the resultant tar archive into Docker as an image, adding Docker specific meta-data like exposed ports (a necessity that should be removed soon when Packer builds this feature in natively) to the final application image, and using supervisorctl to restart the mmckeen.net container, putting into action all the changes in the new build.

Of course, all of the above could have been done just as simply by using Docker and Dockerfiles directly, but here comes the amazing thing about Packer: its flexibility.

Vagrant
-------

Vagrant is the quintessential tool these days for isolating developer environments.  I also use it for testing of software projects, particularly in situations where a full VM is easier to use/provision than a Docker image, or I am working with others who may be on Windows or Mac systems.  Say my website was a much more substantial software project, wouldn't we want the same configuration of our application in the Docker image to be testable in a virtual machine by developers or QA personnel on non-Linux desktops and laptops?  With Packer, it is a simple addition to our original build template.

```
{
  "builders": [
    {
      "type": "docker",
      "image": "mmckeen/opensuse-13-1",
      "export_path": "mmckeen.net.tar"
    },
    {
      "type": "virtualbox-ovf",
      "source_path": "./openSUSE_13.1_Packer_Base-1.0.0/openSUSE_13.1_Packer_Base.x86_64-1.0.0.ovf",
      "ssh_username": "root",
      "ssh_password": "",
      "ssh_wait_timeout": "2m",
      "vboxmanage": [
        ["modifyvm", "{{.Name}}", "--nic1", "nat"]
      ],
      "shutdown_command": "shutdown -P now"     
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "inline": [
                  "zypper -n ref",
                  "zypper -n up",
                  "zypper -n in nginx"
                ]
    },
    {
      "type": "shell",
      "only": ["docker"],
      "inline": [
                  "echo 'daemon off;' >> /etc/nginx/nginx.conf"
                ]
    },
    {
      "type": "shell",
      "only": ["virtualbox-ovf"],
      "inline": [
                  "useradd vagrant",
                  "echo 'vagrant    ALL = NOPASSWD: ALL' > /etc/sudoers",
                  "mkdir -p /home/vagrant/.ssh",
                  "wget --no-check-certificate -O authorized_keys 'https://github.com/mitchellh/vagrant/raw/master/keys/vagrant.pub'",
                  "mv authorized_keys /home/vagrant/.ssh/",
                  "chown -R vagrant /home/vagrant/.ssh"
                ]
    },
    {
      "type": "file",
      "source": "/srv/www/mmckeen.net/",
      "destination": "/srv/www/htdocs/"      
    }
  ],
  "post-processors": [
    {
      "type": "vagrant",
      "only": ["virtualbox-ovf"],
      "output": "mmckeen_net_virtualbox.box",
      "vagrantfile_template": "./Vagrantfile.template"
    } 
  ]
}
```

Of course simple might not be a 100% accurate term, but if you look beyond the code required as setup for basically any Vagrant box, setting up Vagrant's SSH key and sudo access, the difference between the two templates only comes from two sections: post-processors and the second builder object of type virtualbox-ovf.

```
    {
      "type": "virtualbox-ovf",
      "source_path": "./openSUSE_13.1_Packer_Base-1.0.0/openSUSE_13.1_Packer_Base.x86_64-1.0.0.ovf",
      "ssh_username": "root",
      "ssh_password": "",
      "ssh_wait_timeout": "2m",
      "vboxmanage": [
        ["modifyvm", "", "--nic1", "nat"]
      ],
      "shutdown_command": "shutdown -P now"     
    }
```

```
  "post-processors": [
    {
      "type": "vagrant",
      "only": ["virtualbox-ovf"],
      "output": "mmckeen_net_virtualbox.box",
      "vagrantfile_template": "./Vagrantfile.template"
    } 
  ]
```

Using a OVF virtual machine image that I created using the SUSE Studio service (http://susestudio.com/a/ZNpZV4/opensuse-13-1-packer-base), Packer imports the machine image into VirtualBox, logs in over SSH, and executes the commands outlined in the provisioners section of the build template.  After doing this the Vagrant post-processor takes over, bundling the VirtualBox VM along with a template Vagrantfile to form a Vagrant box containing all the code, system services, and configuration to run mmckeen.net in its own dedicated virtual machine using Vagrant.

Conclusion
----------

At this point we have one Packer template that can build both a Docker and Vagrant Box concurrently to run mmckeen.net, or either one separately.  On my Digital Ocean VPS I use the Docker builder to generate each release of this website, and on my local machine, the VirtualBox builder and Vagrant post processor to generate a Vagrant Box for local testing, and use on systems where I don't have Docker handy.  Of course the configuration necessary to run this website is extremely minimal, and I don't tend to release new versions very often, but the power of Packer's approach comes with complex configuration driven by configuration management, continuous testing and integration tooling, and I think is a completely awesome way to fuel the construction of multiple or even single approaches to deploying, testing, and developing various types of applications.

As always, all the code used in this post, with the exception of the deployment script, is available from my Github [packer-build-templates](https://github.com/mmckeen/packer-build-templates) repository.
