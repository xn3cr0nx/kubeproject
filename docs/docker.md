# Docker

Docker is a containerization technology that enables the creation and use of Linux containers. With Docker, you can treat containers like extremely lightweight, modular virtual machines. And you get flexibility with those containers you can create, deploy, copy, and move them from environment to environment.

## Linux Containers

A Linux container is a set of processes that are isolated from the rest of the system, running from a distinct image that provides all files necessary to support the processes. By providing an image that contains all of an application’s dependencies, it is portable and consistent as it moves from development, to testing, and finally to production. This is why containers are extensively used by teams.

### Virtualization

The difference from the virtualization is that with the virtualization many operating systems run simultaneously on a single system. Containers share the same operating system kernel and isolate the application processes from the rest of the system. This means that the software that makes virtualization work, isn’t as lightweight as using containers.

The truth is that the Docker technology brings more than the ability to run containers, it also eases the process of creating and building containers, shipping images, and versioning of images (among other things). That's why Docker containers are different from the original concept of Linux container.

## How it works

The Docker approach is the SOA (service oriented architecture) e.g. a microservices based approach.

### Layers

Each Docker image file is made up of a series of layers. These layers are combined into a single image. A layer is created when the image changes. Every time a user specifies a command, such as run or copy, a new layer gets created. Docker reuses these layers for new container builds, which makes the build process much faster. Intermediate changes are shared between images, further improving speed, size, and efficiency.

## Advantages

* Rapid application deployment: containers include the minimal runtime requirements of the application, reducing their size and allowing them to be deployed quickly.
* Portability across machines: an application and all its dependencies can be bundled into a single container that is independent from the host version of Linux kernel, platform distribution, or deployment model. This container can be transfered to another machine that runs Docker, and executed there without compatibility issues.
* Version control and component reuse: you can track successive versions of a container, inspect differences, or roll-back to previous versions. Containers reuse components from the preceding layers, which makes them noticeably lightweight.
* Sharing: you can use a remote repository to share your container with others. It is also possible to configure your own private repository.
* Lightweight footprint and minimal overhead: Docker images are typically very small, which facilitates rapid delivery and reduces the time to deploy new application containers.
* Simplified maintenance: Docker reduces effort and risk of problems with application dependencies.

All Rights Reserved © 2018 Red Hat, Inc.