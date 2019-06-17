---
title: "Docker Import and Push Support for Packer"
date: 2014-01-20T16:56:31-07:00
coverImage: /images/import-export.png
coverMeta: out
coverSize: partial
thumbnailImagePosition: right
tags:
- docker
- packer
categories:
- containerized deployment
- code contributions
---

Over the last few weeks I've put together a pull request for Packer which should be releasing soon with version 0.5.2.

With my usage of Packer and Docker, I've always found it an annoyance to have to import the Packer built Docker
image separately, using Docker import, rather than have Packer handle importing with a post-processor.

Once 0.5.2 releases, the deployment of a Packer built Docker image can be optimized through the use of the docker-push and docker-import
post processors, through a build template with post-processors much like the following:


```
{
  "type": "docker-import",
  "repository": "mmckeen/packer",
  "tag": "0.5.2"
},
"docker-push"
```

These post processors will automatically import the generated Packer artifact into the Docker daemon and push it to a remote repository,
simplifying a good majority of the deployment script from my previous blog [post](http://mmckeen.net/blog/2013/12/27/advanced-docker-provisioning-with-packer/).
