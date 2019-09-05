---
export_on_save:
  html: true

html:
  embed_local_images: false
  embed_svg: true
  offline: true
  toc: undefined

print_background: false

---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [A brief introduction to Docker](#a-brief-introduction-to-docker)
    - [A few definitions](#a-few-definitions)
  - [The benefits of containers](#the-benefits-of-containers)
  - [Hello world!](#hello-world)
  - [Images](#images)
    - [Docker Hub](#docker-hub)
    - [Creating images](#creating-images)
  - [Walking before `run`ning](#walking-before-running)
    - [Naming containers](#naming-containers)
    - [Interacting with containers](#interacting-with-containers)
  - [Cleaning up after yourself](#cleaning-up-after-yourself)
  - [Dockerfiles](#dockerfiles)
    - [CMD vs RUN vs ENTRYPOINT](#cmd-vs-run-vs-entrypoint)
    - [Shell form vs exec form](#shell-form-vs-exec-form)
    - [Backslashes in shell syntax](#backslashes-in-shell-syntax)
    - [Best practices and efficient images](#best-practices-and-efficient-images)
    - [Background processes](#background-processes)
  - [Managing data](#managing-data)
  - [When things go wrong](#when-things-go-wrong)
  - [Docker-compose](#docker-compose)
- [Resources](#resources)

<!-- /code_chunk_output -->

# A brief introduction to Docker

Docker is an open-source containerization tool. It allows you to package and run an application (or even just a script or a simple command) in a _container_, an isolated sandbox that runs on your host (local) operating system (OS). The container will not only contain the application itself, but also all of its dependencies, including a (stripped-down) operating system.

Containers are often used for deploying application on a server, since they are straight-forward to migrate and scale-up, but they can also be a real asset in reproducible science! By providing a docker image of your application or even just an analysis pipeline, others will be able to run the code in the exact same environment you originally implemented it.

![](assets/containers.jpeg)

They are similar to virtual machines (VMs), but there are two distinguishing features that set them apart ([see official docs](https://docs.docker.com/get-started/#containers-and-virtual-machines)). First of all, from a technical standpoint, containers perform fewer emulation tasks since they share the host OS' kernel.[^1] A process running in a container will use the same amount of memory as a native process. VMs on the other hand rely on virtualized hardware to run a completely isolated guest OS. Secondly, while virtual machines are often used in an interactive manner, by providing the user with a full OS-in-a-box that they can use just like a standard desktop, containers are intended to run as one-off instances, but more on that later.


<!-- This makes them more resource-hungry (and slower?), although [todo cost of isolation security].  -->

### A few definitions

- host machine: this is the operating system that is running the docker service and launching containers. It might be your own computer, a web server in the cloud or even another docker process.
- container: a temporary environment, based on an image, which runs an application.
- image: a snapshot or blueprint of an application or system, containing an operating system, dependencies, applications, files, background processes, etc. Containers are created from an image state. Images are built on top of each other in a hierarchical manner.

## The benefits of containers

- **Reproducibility**
    - across operating systems (with some caveats [^1])
    - dependency management
    - it works on my machine
![](assets/mymachine.jpg)
- Running software/scripts/commands **without installing** them or any of their dependencies manually
    - no need to dig through poor installation documentation or debug obscure compilation errors
    - long lists of required dependencies are things of the past
    - software and dependencies you don't plan to use often will not pollute your system library
    - no installation privileges required
- Comparing **different versions** of software side-by-side (or testing your software in different environments)
- Easier transition from **development to production** environment in the case of web applications
    - local and production environment _should_ behave exactly the same.

## Hello world!

We'll first create a very simple docker application, and then delve into its components step by step.

```
docker run -it --rm debian echo 'Hello world!'
```

```
Hello world!
```

Congrats, you've just run your first docker container!

Let's unpack what has happened here:

1) `docker run` attempts to launch a container.
2) The container is based on the image named `debian`.
3) `-it` is a flag that tells docker to redirect the container's input/output/error streams to our host's terminal. `--rm` removes the container after it has finished running.

Don't worry though, we'll go over everything in more detail step by step. Starting with...

## Images

Containers, the isolated instances that run a specific application, are always based on an image. You can think of the image as a **blueprint** or a system snapshot. It defines a system environment, any required libraries, the file system and (configuration) files, and even things like background processes or  necessary preparatory steps. In a nutshell, all the steps we would have to perform on our own operating system before we can use the desired application.

### Docker Hub

The [Docker Hub](https://hub.docker.com/explore/) is an only repository where you can browse images made by both regular users and so called [_Official Images_](https://docs.docker.com/docker-hub/official_images/), a curated set generally consisting of base operating systems (e.g. [debian](https://hub.docker.com/_/debian), [ubuntu](https://hub.docker.com/_/ubuntu) and [busybox](https://hub.docker.com/_/busybox)[^2] and ready-to-go containers for popular frameworks and languages (e.g. [python](https://hub.docker.com/_/python), [R](https://hub.docker.com/r/rocker/r-ver), [nodejs](https://hub.docker.com/_/node/)).

### Creating images

Images are hierarchical. Most of the images you will create yourself will be based on a previous, more general parent image, often one of those Official Images, but perhaps in a certain situation it makes more sense to base an image on one of your own.

You can list all of the images that are available on your system as follows:

```bash {cmd=true}
docker image ls
```

[todo]
<small>You might notice a bunch of duplicates tagged as `<none>`. Don't worry, these images only take up space once.</small>

And you can obtain a specific image manually using the `docker pull`. The names are formatted like  [todo],, version. Many images allow you to either specify a version or just use the latest available (stable) version (often labeled `latest` or `stable`). Some images also provide a `slim` version, these are usually light-weight images lacking most functionality apart from only those things that are required to run a specific application. These are a good bet if you want to reduce the file size of your images (but more on that later).

Most of the time however, you won't be using the `pull` command yourself, instead, you'll either run a container based on an image directly and docker will download the required image automatically (as in the _Hello World_ example) or you will build your own image based on a `Dockerfile`. We'll go over those in a bit, but let us first dive into `run` statements and existing images.

## Walking before `run`ning

Let us explore the `docker run` command in some more detail, without worrying about _what exactly it is that we're running_.

The simplest form of a run command would be something along the lines of the following:

```bash {cmd=true}
docker run debian
```

Which produces no output... Huh? What's going on here?

Well, the above command spins up a container that is based on the base `debian` image. You can look up details (and the `Dockerfile`) of this image [here](https://hub.docker.com/_/debian). You will see that there are multiple versions of this image, so-called _tags_. It's a good practice to always specify the correct version tag of images, since a tag like _latest_ is not static in time.

This image has a command that specifies the process it will run, which is simply: `CMD ["bash"]`.

When we issue the above command, behind the scenes a container will start, and our supplied command (which is empty) will be passed on to this `bash` statement. Having nothing to do, the container will simply shut down itself immediately.

Instead, we can pass an argument like we did in the _Hello World_ example.

```bash {cmd=true}
docker run debian echo "The current working directory is $(pwd)" \
                    && cd /var/ \
                    && echo "the contents of /var/ are" \
                    && ls -l
```

<small>If you're not familiar with the `&&` syntax, it is a way to chain multiple commands into one statement. Each subsequent command will only be executed after the previous one is finished (succesfully).</small>

To see a list of active containers, the `docker container ls` command can be used. What do you think we'll see when we run it at this point?

```bash {cmd=true}
docker container ls
```

If you guessed nothing, you were right. The list is still empty because the container shut down itself after running the final `ls -l` command. Containers do their job and then shut down.

There is however a more useful variant of the `container ls` command that we can run:

```bash {cmd=true}
docker container ls --all
```

The `STATUS` column tells us when and how these containers exited (an exit status of 0 in linux means that a command was run without any problems arising).

All of this brings us to one of the most important aspects of docker containers.

> **Docker containers are ephemeral.**

Containers are short-lived. They are created after a specific state of a system (described in the image), will do whatever it is you instructed them to do, and then disappear. They will leave no traces.

We can demonstrate this behaviour by running two containers in a row.

```bash {cmd=true}
docker run debian tee 'cowabunga!' /home/pizza
docker run debian ls -l /home
```

_Always keep this in the back of your mind. It will be extremely important when you start relying on containers to produce (and hopefully store) the output of a certain analysis._

### Naming containers

Docker containers are given an ID automatically (e.g. `7a0c1129df9a`), but this is rather cumbersome. Instead, you can use the `--name` flag to specify your own name to refer to a specific container.

### Interacting with containers

Most of the time we want to run a container in the background, but there are a few situations where interaction can be useful. Debugging is one of them. Trying out small parts of an image you are creating is another.

The easiest way is to pass the `-it` flag (allocates a tty) and specify a shell command such as `sh` (for the minimal `busybox` images) or `bash` (in most of the linux-based images). Give it a try!

    docker run -it debian bash

<small>Use `<ctrl>+d` or type `exit` to leave (and shut down) the container.</small>

This is quite useful whenever you're unsure of the exact file structure your image has. I.e., you can write a Dockerfile, add some files, then run it interactively to see if everything is where it's supposed to be. In a similar vein, I've found it useful to figure out file permissions and user ids.

If you're feeling adventurous, this also allows you to finally witness what happens when you run a command like `rm --rf`. I'm not liable when you accidentally run it in your host terminal instead of the container though!

Another method is provided by the `exec` command ([see docs](https://docs.docker.com/engine/reference/commandline/exec/)), which can inject a command into any containers _that is already running_. For example:

```bash
docker run -it --rm --name sleepy_debian -d debian sleep 100
docker container ls
docker exec -it sleepy_debian bash
```

Note that we used `--name` to give the container a name, rather than relying on the long ID, and we used the `-it` flag again to enter interactive mode.

## Cleaning up after yourself

To clean up all these unused containers you have collected (see `docker container ls --all`), you can use the `prune` commands:

`docker container prune`

Or you can issue the `--rm` flag to your `run` statements. This will ensure that containers are removed after they shut down.

There also exists a prune command for images (and other docker entities), so have a look at the docs [https://docs.docker.com/config/pruning/](https://docs.docker.com/config/pruning/) for more information on how these pruning commands behave.

## Dockerfiles

Dockerfiles are the meat and potatoes of docker. All the images we have used so far, are based on a Dockerfile as well. You can inspect it by clicking on the tag names in Docker Hub.

Rather than relying on pre-existing images, we will build our own. Here is an example:

    # Use an official Python runtime as a parent image
    FROM python:slim

    # Ensure that Python outputs everything that's printed inside # the application rather than buffering it.
    ENV PYTHONUNBUFFERED 1

    # Set the file maintainer (your name - the file's author)
    MAINTAINER Splinter

    # Set environment variables
    ENV MY_DIR=/opt/technodrome

    # ARG variables are only available during the build process
    ARG USERID=999
    ARG GROUPID=999

    # Copy a pip requirements file, install packages and remove the leftovers
    COPY requirements.txt /tmp/
    RUN pip install --no-cache-dir -r /tmp/requirements.txt && \
        rm -rf /tmp/requirements.txt

    # Install required packages via apt and clean up
    RUN apt-get update \
        && apt-get install -y --no-install-recommends \
            cowsay \
        && rm -rf /var/lib/apt/lists/*

    # Create a non-root user (optional!)
    RUN groupadd -r -g ${GROUPID} tmnt && \
        useradd -r -g Donatello -u ${USERID} -s /sbin/nologin Donatello

    # Make port 8000 available to the world outside this container
    EXPOSE 8000

    # Copy the current directory contents into the container at /app
    COPY . /opt/src

    # Copy source (and data files)
    RUN echo "important ninja business" > ${MY_DIR}/ooze.txt

    # Remove windows new lines and set correct permissions
    # NOTE: initialize empty directory for shared static volume!
    #       otherwise the permissions will be lost!
    RUN sed -i 's/\r//' /opt/src/scripts/* && \
        chmod -R +x /opt/src/scripts && \
        chown -R Donatello /opt/src

    # Change to non-root user
    USER Donatello

    # Set the working directory
    WORKDIR /opt/src

    # Entrypoint to perform Django static file collection
    ENTRYPOINT ["bash", "./scripts/pizza-time.sh"]

    # Run your script
    # CMD ["./scripts/shell-shocked.sh"]


### CMD vs RUN vs ENTRYPOINT

The difference between these is explained in more detail [here](https://goinbigdata.com/docker-run-vs-cmd-vs-entrypoint/) and [here](https://aboullaite.me/dockerfile-run-vs-cmd-vs-entrypoint/).

In a nutshell:

- `RUN` statements are used to configure things inside the image, e.g. installing software. The state of the final image is post-these-operations.
- The `CMD` statement defines the command that is executed by default when a container is created from the image. Note however, that you can override this value by passing another command via the command line, e.g. `docker run image <command-here>`
- `ENTRYPOINT` statements cannot be be (easily) overridden from the command line. They should be used 

### Shell form vs exec form

Inside a Dockerfile (and also [compose files](#docker-compose)) there are two forms of defining commands (for `ENTRYPOINT`, `CMD` and `RUN` statements).

- exec form: `ENTRYPOINT ["executable", "param1", "param2"]`
- shell form: `ENTRYPOINT command param1 param2`

The former is [prefered by the docs](https://docs.docker.com/engine/reference/builder/#entrypoint), and more information can be found [here](https://nickjanetakis.com/blog/docker-tip-63-difference-between-an-array-and-string-based-cmd) and [here](https://gmaslowski.com/docker-shell-vs-exec/).

However, the exec form does not perform all types of shel processing, e.g. variable expansion will not occur (because the statement is processed by docker instead of the shell inside the container). [See here](https://docs.docker.com/engine/reference/builder/#run).

### Backslashes in shell syntax

You can use a `\` to split a command over multiple lines:

    RUN pip install --no-cache-dir -r /tmp/requirements.txt && \
        rm -rf /tmp/requirements.txt

### Best practices and efficient images

Poorly written Dockerfiles can lead to oversized images, so I recommend having a look at [some best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#user). A major remark is to keep the number of statements (or layers) to a minimum. You can do this by combining multiple `RUN` statements into one:

    RUN apt-get update && apt-get install -y \
        aufs-tools \
        automake \
        build-essential \
        curl \
        dpkg-sig \
        libcap-dev \
        libsqlite3-dev \
        mercurial \
        reprepro \
        ruby1.9.1 \
        ruby1.9.1-dev \
        s3cmd=1.1.* \
    && rm -rf /var/lib/apt/lists/*

### Background processes

Sometimes you want processes to run in the background, i.e. you do not want all the output to appear on your terminal. To do that, you can use the `-d/--detach` flag.

## Managing data

You've seen that you can use a `COPY` statement to get files into your container. And you also know that containers are temporary. So does that imply you lose all of the output your process generates?

That brings us to _volumes_. A complete overview is given in [the docs](https://docs.docker.com/storage/volumes/).

There are two types of volumes: bind mounts and named volumes.

With bind mounts, you mount a directory on your host machine on top of a directory inside of your container.

```bash {cmd=true}
mkdir -P bind_mount_test
touch bind_mount_test/krang
docker run -it --rm -v /my/local/dir:/container/dir
```

What is important to take note of here is that the bind mount will override anything that is in your container. For example, suppose we have a statement like `COPY . /src` in our Dockerfile (which would initialise a directory in the image located in `/src` containing all files in your local working directory). However, when you bind mount something on top of `/src`, it will override its contents. This ensures that you never accidentally wipe data on your host machine by replacing it with things inside an image. Remember, the image is self-sufficient and contains all information it would ever require, your host machine is not.

Also note that the first filepath (the one on the host machine), cannot be a relative path, it must be the full filepath.

Named volumes are the second type of volume. Instead of mounting a host directory, they simply give a name to a directory inside of your container. After the container is finished (and/or removed), the volume will remain. You can then later mount it again or retrieve data from it directly. They use the same syntax as bind mounts, but replace the first filepath into a name, e.g. `-v my_volume:/my/container/path`.

Finally, there is also an option to copy data directly into a _running_ container: [https://docs.docker.com/engine/reference/commandline/cp/](https://docs.docker.com/engine/reference/commandline/cp/).

## When things go wrong

When things go wrong, and trust me, they almost always do, you will need some tricks to figure out what's going on. Like always, google is your best friend in this regard. Docker's documentation is also quite extensive and I highly recommend you consult it when trying out new features. That being said, it will likely take some time and experience before you will be able to completely digest everything and understand all implications that a certain explanation might have on seemingly unrelated aspects of Docker.

Unfortunately, it is likely you will hit a few bumps when you attempt to configure your first containers (or set of containers).

## Docker-compose

Generally, containers should only be doing one thing. There might be a few background processes running (indeed, there's usually an entire linux OS running), and there's nothing wrong with running multiple scripts in a row (the script calling the pipeline itself can be considered the _"one thing".), but as soon as you find yourself trying to run multiple things simultaneously, you should reach for `docker-compose`.

Docker-compose allows you to orchestrate multiple containers as one unit. This is useful for things like web applications that require a web server, load balancer, database, etc.

Docker-compose is managed by `docker-compose.yml` files, which are YAML files that borrow most of their syntax from Dockerfiles. Most options can be set in either, but off-loading most of the work to compose files is useful because you'll be able to re-use parts across multiple containers. You can also specify `docker run` command line flags directly in compose files, making it much easier to set up a shared volume across a bunch of containers or to run multiple containers in the same virtual network.

For a more complete overview of how docker-compose works, check out the [documentation](https://docs.docker.com/compose/).

# Resources

- The official docs: https://docs.docker.com/
- A superb tutorial that walks through everything I've covered (and more) in a very digestable format, highly recommended! https://docker-curriculum.com/
- HandsOnDocker: self-paced labs. https://github.com/alexellis/handsondocker/
- Best practices for writing Dockerfiles: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- Common mistakes:
    - https://blog.developer.atlassian.com/common-dockerfile-mistakes/
    - https://runnable.com/blog/9-common-dockerfile-mistakes

[^1]: Technically this is only the case when you run docker on Linux. In Windows and MacOS, docker runs inside a Hyper-V VM, which still only utilises part of your system's hardware through virtualization.

[^2]: Busybox is a tiny image that provides various UNIX utilities. Note that it does not even contain `bash`, only `sh`.

<!--

TODO

container ls --all

deleting containers

system pruning

down/stop remove container or not?

name containers

image size

permissions

compose -> yaml anchors

## Practicalities

A few notes about installing and setting up docker. Permission denied => sudo group

safety => sudoer certain commands

docker-engine vs ...


## Updating data in a volume

see joplin

 -->
