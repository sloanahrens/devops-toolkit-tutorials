# Part 6: Building and maintaining a Kubernetes cluster from CI

### Complete Parts 1-5

Make sure you've completed the first five parts of the tutorial.

I'll assume that you still have all the files in place from the previous three exercises, contained in your `source` directory at the top level of your local clone of the `devops-toolkit` [repostitory](https://github.com/sloanahrens/devops-toolkit).

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

In this exercise we'll use the `devops` development environment (rather than the `baseimage` dev env from Parts 2 and 3), so make sure you can run it as described [here](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/0-local-dev-env-devops.md).

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

Run `ls` from inside the container and should see the files you created in the `source` directory:

```
root@1bc94b877039:/src# ls
container_environments	django	docker
```

### Kubernetes and Rancher
...

[Prev: Part 5](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-5-ci-circleci-aws.md)
[Next: Part 7]()
