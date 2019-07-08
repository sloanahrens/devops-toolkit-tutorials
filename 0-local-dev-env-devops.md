# Part 0: Local DevOps Docker Development Environment

There are two different development environments used in the tutorials, the `devops` environment and the `baseimage` environment.
For both, you will need Docker and `git` installed in your host OS.


### Install Docker

If you do not have Docker installed on your system, you will need to install the appropriate version for your operating system.
Here are [instructions for Mac OSX](https://docs.docker.com/docker-for-mac/install/).

You can check that Docker is working by running:

```bash
docker ps
```

It probably won't show you anything, because we haven't started any processes yet, but you should see this at least:

```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

### Set up `git` and GitHub

If you do not already have `git` installed on your system, you will need to install it. 
We're going to use a running docker container as our development environment, with code in a [docker volume](https://docs.docker.com/storage/volumes/) acting as a sort of "shared folder", which is a directory shared by the host and the running docker container. 
This accomplishes a couple of things. 
One, it makes it a lot easier to use a text editor of your choice from your host OS.
Two, it means that if we build a bunch of code inside the dev environment and then delete the docker container, we haven't lost our work.

You will also need a [GitHub](https://github.com) account set up, with a working SSH key if you want to use one (setup instructions can be found [here](https://help.github.com/en/articles/set-up-git)). 
I prefer authenticating with an SSH key but it doesn't matter that much. 

Once you have `git` installed, you will need to get it [configured with your name and email](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup).

### Clone `devops-toolkit` repository

With `git` installed and configured, and your Github account set up, you can run:

```bash
git clone git@github.com:sloanahrens/devops-toolkit.git  # SSH
```

or

```bash
git clone https://github.com/sloanahrens/devops-toolkit.git  # HTTP auth
```

and then navigate to the new directory with:

`cd devops-toolkit`


### `devops` image development environment

The Docker image defined in the [`devops` Dockerfile](https://github.com/sloanahrens/devops-toolkit/blob/master/docker/devops/Dockerfile) can be used for local development for devops-related tasks. 

First we need to build and tag the `devops` image.
From the root of your local clone of the `devops-toolkit` repository, run:

```bash
docker build -t devops -f docker/devops/Dockerfile .
```

So that command builds a [Docker image](https://docs.docker.com/engine/reference/commandline/images/).
As homework later, you should go read some about what Docker is and how it works.
These tutorials largely just explain how to use it to do specific things.

Then we can run our new Docker image with:

```bash
docker run -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD:/src \
  --rm \
  --name \
  local_devops \
  devops \
  /bin/bash
```

Once you run the command, you should see a prompt like (the ID will be different):

```
root@13e70f76d2b8:/src#
```

That is your "container prompt", the

*Aside*: For reference, Here is an annotated version of that last command (pasting it probably won't work because of the comments):

```bash
# if needed replace $PWD with full path to local devops-toolkit folder
docker run -it \  # using -it creates an interactive session
  -v /var/run/docker.sock:/var/run/docker.sock \  # use host's docker engine inside container
  -v $PWD:/src \  # share local directory on host and container
  --rm \  # remove the container when finished (upon exit)
  --name local_devops \  # name the container for ease of use 
  devops \  # devops image should be built and tagged
  /bin/bash  # start a bash terminal
```

From your container prompt (`root@whatever:/src#`), type `docker ps` and hit enter and you should see something like:

```
root@13e70f76d2b8:/src# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
13e70f76d2b8        devops              "/bin/bash"         52 seconds ago      Up 51 seconds                           local_devops
```

Which is to say, you should see the container inside of which we are currently working, from the container itself! 
Notice that the `CONTAINER ID` matches the one in your `root@...` prompt inside the running container.

Type `ls` and you should see the contents of the `devops-toolkit` `git` repo, thanks to our docker volume:

```
root@ae87cc98313a:/src# ls
README.md  ci_scripts  container_environments  django  docker  kubernetes  tutorials
```

To start a second session (in another terminal tab or [screen](https://help.ubuntu.com/community/Screen), perhaps) into the already-running container, you can run:

```bash
docker exec -it local_devops /bin/bash
```

To exit the environment, just type `exit`.

If you get an error message like:

```
docker: Error response from daemon: Conflict. The container name "/local_devops" is already in use by ...
```

then you can remove the existing container with:

```bash
docker rm local_devops
```

and then run the `docker run` command again.

When you're finished working in the `local_devops` container simply type `exit`.

[Next Part 1](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-1-microservices-django.md)
