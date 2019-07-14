# Part 5: Creating a Kubernetes cluster with `kops` and updating it from CI

### Kubernetes and Kops

If you've done any sort of DevOps work lately, or probably software-development in general, you have likely heard of [Kubernetes](https://kubernetes.io/).
Kubernetes is not the only [container orchestration system](https://blog.newrelic.com/engineering/container-orchestration-explained/), but it's become the most important.
Here are some [fun K8s facts](https://dzone.com/articles/10-basic-facts-about-kubernetes-that-you-didnt-kno).

I'm not going to concentrate on teaching you about Kubernetes, but rather demonstrate how to deploy our application to a Kubernetes cluster in the cloud.

Before we can deploy our applicaiton to a K8s cluster, we have to deploy said K8s cluster.
There are lots of ways to do this.
We could use a hosted service like [Amazon's EKS](https://aws.amazon.com/eks/) or [Google's GKE](https://cloud.google.com/kubernetes-engine/).

In this tutorial, however, we are going to use [kops](https://kubernetes.io/docs/setup/production-environment/tools/kops/).
More specifically, we are going to use `kops` to create [Terraform](https://www.terraform.io/) code which we will then use to create our cluster, and check into source control.
This is far from the only way to solve this problem, but it is a useful one for a number of reasons:

- This method works well if you only need a few clusters, and we only need one.
- kops is easy to use, yet makes it easy to rebuild a cluster from scratch if needed.
- The code that defines our cluster can be checked into source control, but we don't have to build it by hand.
- This method works well for learning Kubernetes

There are a couple of downsides to this approach:

- The `kops create` command is not [idempotent](https://en.m.wikipedia.org/wiki/Idempotence) and therefore is difficult to integrate into a CI pipeline.
- Managing more than a handful of clusters quickly becomes difficult and error-prone.

To solve these problems, we could use the powerful set of tools from [Rancher](https://rancher.com/).
I plan to build a Rancher tutorial soon.
For now I want to focus at a slightly lower level.

### Complete Parts 1-4

Make sure you've completed the parts of the tutorial up to this point.

I'll assume that you still have all the files in place from the previous exercises, contained in your `source` directory at the top level of your local clone of the `devops-toolkit` [repostitory](https://github.com/sloanahrens/devops-toolkit).

Your `source` directory should have at least the following structure:

```
devops-toolkit/
    source/
        .circleci/
            config.yml
        container_environments/
            test-stack.yaml
        django/
            stockpicker/
                ...
            requirements.txt
        docker/
            baseimage/
                Dockerfile
            celery/
                Dockerfile
            scripts/ 
                initialize-webapp-prod.sh
                wait-for-it.sh
            stacktest/
                Dockerfile
                integration-tests.sh
            webapp/
                Dockerfile
            docker-compose-local-dev-django.yaml
            docker-compose-local-dev-django-celery.yaml
            docker-compose-local-dev-django.html-celery-beat.yaml
            docker-compose-local-image-stack.yaml
            docker-compose-tagged-image-stack.yaml
            docker-compose-unit-test.yaml
        .dockerignore
        .gitignore
```

### `devops` Development Environment

In this exercise we'll use the `devops` development environment, so make sure you can run it as described [here](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/0-local-dev-env-devops.md).

Make sure you have built the `devops` Docker image, by running the following command from your `devops-toolkit` directory:

```bash
docker build -t devops -f docker/devops/Dockerfile .
```

Go to your `source` directory now, and and start the development environment Docker container with:

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

Remember that in this command, the argument `-v $PWD:/src` connects the directory on the host OS from which you _run_ the command, to the `/src` directory inside the container.

Run `ls` from inside the container and should see the files you created in the `source` directory:

```
root@1bc94b877039:/src# ls
container_environments	django	docker
```

### `kops create`



[Prev: Part 4](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-4-ci-circleci-aws.md)
[Next: Part 6]()
