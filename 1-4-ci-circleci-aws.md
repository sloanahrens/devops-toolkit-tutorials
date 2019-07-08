# Part 4: CI: Automate Building App Deliverables with GitHub, CircleCI, and AWS

In this exercise we will take what we've built so far and automate it by building a true continuous-integration pipeline.
The deliverables of this pipeline will be our application docker images, automatically pushed to a private Docker repository under our control.
We will need to set up some configuration to hook all the pieces together and get the pipeline working.
In later exercises we will build out more CI modules as we build up more DevOps infrastructure.

### Sign up for stuff

A consequence of doing a concrete, step-by-step tutorial is that some concrete decisions have to be made about service providers.
I made the choices that I believe are the most sensible defaults.
This means that to follow along you will have to commit to signing up for a few things.
It shouldn't cost you much if any money, if you're careful.

_But it might cost actual money so be careful._

I'm going to ask you to sign up for accounts with [GitHub](https://github.com/), [CircleCI](https://circleci.com/), and [Amazon Web Services](https://aws.amazon.com/).

There are--as is often the case in DevOps--suitable alternatives to all these choices.
You could use [Bitbucket](https://bitbucket.org/product/) instead of GitHub.
You could use [Jenkins](https://jenkins.io/) instead of CircleCI, either [hosted](https://duckduckgo.com/?q=hosted+jenkins) or self-maintained, or any of countless other [CI tools](https://duckduckgo.com/?q=top+continuous+integration+tools).
You could use [Azure](https://azure.microsoft.com/en-us/) or [Rackspace](https://www.rackspace.com/) instead of AWS.
It would be a worthy exercise to pick some different providers and rebuild portions of this tutorial with those instead of my choices.
 
### Warning: AWS costs $$$

Be careful with your AWS usage.
I'll bring this point up again, but Amazon will happily bill you for all your cloud infrastructure use, so pay attention to it.

I won't ask you to anything that will cost more than a few dollars a month, if that--at least until we get around to deploying [Kubernetes](https://kubernetes.io/).


### Complete Parts 1-4

Make sure you've completed the first four parts of the tutorial.

I'll assume that you still have all the files in place from the previous three exercises, contained in your `source` directory at the top level of your local clone of the `devops-toolkit` [repostitory](https://github.com/sloanahrens/devops-toolkit).

Your `source` directory should have at least the following structure:

```
devops-toolkit/
    source/
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
            docker-compose-unit-test.yaml
```

### `devops` Development Environment

In this exercise we'll use the `devops` development environment (rather than the `baseimage` dev env from Parts 2 and 3), so make sure you can run it as described [here](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/0-local-dev-env-devops.md).

Make sure you have built the `devops` Docker image, by running the following command from your `devops-toolkit` directory:

```bash
docker build -t devops -f docker/devops/Dockerfile .
```

Go to your `devops-toolkit/source` directory now, and and start the development environment Docker container with:

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

### Create a GitHub repository

At this point, we have to create a GitHub repository to continue, because we need to connect CircleCI to the repository.
So make sure you are signed up for [GitHub](https://github.com/), and create a public repository.
You can name it whatever you want.
For the purposes of this tutorial, I named mine `devops-toolkit-test`.

### Set up CircleCI

After creating the repository, we need to connect it to CircleCI.
So make sure you've signed up for [CircleCI](https://circleci.com), connect it to GitHub, and then connect your CircleCI account to your new GitHub repository.
CircleCI currently has a free tier that will be more than adequate for our needs.

### Set up CircleCI config

Now we need to create our [CircleCI config file](https://circleci.com/docs/2.0/configuration-reference/).

Create `source/.circleci/config.yaml`:
```yaml
defaults: &defaults
  docker:
    - image: sloanahrens/devops-toolkit-ci-dev-env:0.0.3
version: 2
jobs:

  image_build_test_push:
    <<: *defaults
    steps:
      - checkout
      - setup_remote_docker:
          version: 17.03.0-ce
      - run:
          name: Build Base Image
          command: |
            set -x
            docker build -t baseimage -f docker/baseimage/Dockerfile .
      - run:
          name: Build Webapp Image
          command: |
            set -x
            docker build -t webapp -f docker/webapp/Dockerfile .
      - run:
          name: Build Celery Image
          command: |
            set -x
            docker build -t celery -f docker/celery/Dockerfile .
      - run:
          name: Build Stack Tester Image
          command: |
            set -x
            docker build -t stacktest -f docker/stacktest/Dockerfile .
      - run:
          name: Run Django Unit Tests
          command: |
            set -x
            docker-compose -f docker/docker-compose-unit-test.yaml run unit-test
            docker-compose -f docker/docker-compose-unit-test.yaml down
      - run:
          name: Run Integration Tests Against Local Docker-Compose Stack
          command: |
            set -x
            docker-compose -f docker/docker-compose-local-image-stack.yaml up -d
            docker run -e SERVICE="localhost:8001" --network container:stockpicker_webapp stacktest ./integration-tests.sh \
                || \
                (echo "*** WORKER1 LOGS:" && echo "$(docker logs stockpicker_worker1)" && \
                echo "*** WORKER2 LOGS:" && echo "$(docker logs stockpicker_worker2)" && \
                echo "*** WEBAPP LOGS:" && echo "$(docker logs stockpicker_webapp)" && \
                exit 1)
            docker-compose -f docker/docker-compose-local-image-stack.yaml down

workflows:
  version: 2
  build-and-deploy:
    jobs:
      - image_build_test_push
```

*WARNING:* A mistake I make over and over is creating `config.yaml` file instead of a `config.yml` file.
It will not work if the extension is `.yaml` and you will pull your hair out trying to figure out why.
So make sure your CircleCI config file is `.circleci/config.yml`.

We also will want to create a [`.gitignore` file](https://git-scm.com/docs/gitignore).
I am currently using the following for the `devops-toolkit` repo:

`source/.gitignore`:
```git
# Created by https://www.gitignore.io

### OSX ###
.DS_Store
.AppleDouble
.LSOverride

# Icon must end with two \r
Icon


# Thumbnails
._*

# Files that might appear on external disk
.Spotlight-V100
.Trashes

# Directories potentially created on remote AFP share
.AppleDB
.AppleDesktop
Network Trash Folder
Temporary Items
.apdisk


### Python ###
# Byte-compiled / optimized / DLL files
__pycache__/
*.py[cod]

# C extensions
*.so

# Distribution / packaging
.Python
env/
build/
develop-eggs/
dist/
downloads/
eggs/
lib64/
parts/
sdist/
var/
*.egg-info/
.installed.cfg
*.egg

# PyInstaller
#  Usually these files are written by a python script from a template
#  before PyInstaller builds the exe, so as to inject date/other infos into it.
*.manifest
*.spec

# Installer logs
pip-log.txt
pip-delete-this-directory.txt

# Unit test / coverage reports
htmlcov/
.tox/
.coverage
.cache
nosetests.xml
coverage.xml

# Translations
*.mo
*.pot

# Sphinx documentation
docs/_build/

# PyBuilder
target/


### Django ###
*.log
*.pot
*.pyc
__pycache__/

.env
db.sqlite3


########## SA
venv
notes.txt
pg_data
celerybeat-schedule
.idea
*.pem
.terraform
kubecfg.yaml
.vagrant
source/
source_first/
_static/
```

It also becomes handy to have a [`.dockerignore` file](https://docs.docker.com/engine/reference/builder/).

Create `source/.dockerignore`:

```
.circleci
ci_scripts
django/venv
kubernetes
kube-prometheus
tutorials
source
source_first
src
_static
```

### Push code

Now we can attach our existing `source` folder to the new GitHub repository (remember to replace `sloanahrens` with your username, and `devops-toolkit-test` with the name of your new GitHub repo):

```bash
cd source
git init
git commit -m "first commit"
git remote add origin git@github.com:sloanahrens/devops-toolkit-tutorials.git
git push -u origin master
```

Now go visit your CircleCi dashboard page.
For me, the URL for the test repo is "https://circleci.com/gh/sloanahrens/devops-toolkit-test", but yours will be different.
Click on the job that should be running, and if you watch the output it should look familiar.
All the same commands are running in the job that we have been running by hand.

Once the job completes, you should see something very similiar to this screenshot (your job ID might be different):

![alt text](https://github.com/sloanahrens/devops-toolkit-tutorials/raw/master/images/circleci_first_job.png "CircleCI job output")

### AWS ECR Docker image repositories

The next step we will need to take is to create a couple of Docker image repositories.
Once you have set up your AWS account, you can create ECR repositories at: [https://us-east-2.console.aws.amazon.com/ecr/repositories](https://us-east-2.console.aws.amazon.com/ecr/repositories).
My default region is `us-east-2`, so that's where I created my ECR repos.
It doesn't matter which region you use, as long as you are consistent.
We need to create two ECR repositories, named `stockpicker/webapp` and `stockpicker/celery`.
We will push docker images to these repos shortly.

The URI for your repos will look similar to this:

```
421987441365.dkr.ecr.us-east-2.amazonaws.com/stockpicker/webapp
```

With different repo ID and region-name depending on your region.
You will need both, so make a note of them.

After you hare created both the `stockpicker/webapp` and `stockpicker/celery` repositories, we should be able to push Docker images to them, once AWS is configured and connected.

### AWS IAM user

In order to push the Docker images we build in CircleCI jobs into our new ECR repositories, we will need to create a user with the proper permissions.
Those permissions are outlined on the README page for this repo, but I'll reproduce them here as well.

The name of the user doesn't really matter, so try to name it something helpful.
Your AWS IAM user needs to be configuired with at least the following permissions:

```
AmazonEC2FullAccess
IAMFullAccess
AmazonEC2ContainerRegistryFullAccess
AmazonS3FullAccess
AmazonDynamoDBFullAccess
AmazonVPCFullAccess
AmazonElasticFileSystemFullAccess 
AmazonRoute53FullAccess
```
So [create your user](https://console.aws.amazon.com/iam/home#/users$new?step=details), like this:

![alt text](https://github.com/sloanahrens/devops-toolkit-tutorials/raw/master/images/aws_iam_create_1.png "Create IAM user")

Make sure you select the "Programmatic access" box.

Next, you can either add the user to a Group, and add all the policies we will need to the Group, or you can simply attach the policies directly to your user. 
I did the latter.

![alt text](https://github.com/sloanahrens/devops-toolkit-tutorials/raw/master/images/aws_iam_create_2.png "Add IAM user permissions")

You can add tags if you want, then review and create the user.
You should see something like:

![alt text](https://github.com/sloanahrens/devops-toolkit-tutorials/raw/master/images/aws_iam_create_3.png "IAM user created")

You will need the `Access key ID` and `Secret access key` values from this page, to plug into CircleCI.
You can copy and paste them from this page, or click the "Download .csv" button to download a CSV file containing the keys. 

### Connect CircleCI to AWS IAM user

Now we need to paste these AWS keys into the appropriate spot in CircleCI.
The CircleCI docs suggest that you use [AWS Orbs](https://circleci.com/docs/2.0/deployment-integrations/#aws-ecr) to accomplish this task.
Since I want this GitHub repository to be public, I cannot add my keys to the repository in a file.

*Note:* If you leave your AWS keys unencrypted and viewable from a public repository, even for a short period of time, you will probably get (like a friend of mine did) [crypto-jacked](https://hackerbits.com/programming/what-is-cryptojacking/).
So don't do that.
Protect your AWS keys like you would protect your most valuable physical keys.

Instead, I added my AWS keys to my CircleCI account in [AWS Permissions environment variables](https://circleci.com/gh/sloanahrens/devops-toolkit-test/edit#aws) in CircleCI.
Just paste your keys into the boxes, and they will be available as environment variables in the CircleCI jobs, which means we can use the [AWS CLI](https://aws.amazon.com/cli/) in our CircleCI jobs.

![alt text](https://github.com/sloanahrens/devops-toolkit-tutorials/raw/master/images/aws_iam_create_3.png "AWS keys in CircleCI")


### Push Docker images

This step isn't strictly necessary, but if you want build and push the production Docker images from your local development machine, here are the steps.

First run the `devops` development environment with:

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

You will need to paste your AWS keys into the environment, as environment variables.
Replace `[REPLACE]` in each of these settings, with your keys:

```bash
export AWS_ACCESS_KEY_ID=[REPLACE]
export AWS_SECRET_ACCESS_KEY=[REPLACE]
```

We will also need two more environment variables.
The first is `ECR_ID`, which you need to get from your ECR repos that you created earlier.
The other is `AWS_REGION`, the region in which you are going to create your assets.
My default region and repo id are:

```bash
export AWS_REGION=us-east-2
export ECR_ID=421987441365
```

The final environment variable we will need is `$IMAGE_TAG`.
There are lots of ways to solve the problem of tagging Docker images in CI.
I like to use the current git branch; it requires a bit of bullet-proofing to work well in CI, but I'll show you how to handle that shortly.
For the moment, let's just use `master` as our value of `IMAGE_TAG`

```bash
export IMAGE_TAG=master
```

Now we are ready to build the Docker images like in Part 4:

```bash
docker build -t baseimage -f docker/baseimage/Dockerfile .
docker build -t webapp -f docker/webapp/Dockerfile .
docker build -t celery -f docker/celery/Dockerfile .
```

But we need to change the tags used for our `webapp` and `celery` images.
We can re-tag our images with the `docker tag` command.
To be compatible with our ECR repos, the tags will need to have the form: `$ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO_NAME:$IMAGE_TAG`.

And so now, the full commands to re-tag the images are:

```bash
docker tag webapp $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/webapp:$IMAGE_TAG
docker tag celery $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/celery:$IMAGE_TAG
```

Before we can push our images, we need to log into AWS with our programmatic user, which we can do with this command:

```bash
$(aws ecr get-login --no-include-email --region us-east-2)
```

You should see "Login Succeeded" in your output:

```
root@60d8f7af06bd:/src# $(aws ecr get-login --no-include-email --region us-east-2)
Login Succeeded
```

Finally, we can push our images to the ECR repos by running:

```bash
docker push $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/webapp:$IMAGE_TAG
docker push $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/celery:$IMAGE_TAG
```

### CircleCI config

With our AWS keys safely tucked away in CircleCI, we can now build and push images to our ECR image repositories from CircleCI jobs.

Update your `source/.circleci/config.yaml` file to match:

```yaml
defaults: &defaults
  docker:
    - image: sloanahrens/devops-toolkit-ci-dev-env:0.0.3
version: 2
jobs:

  image_build_test_push:
    <<: *defaults
    steps:
      - checkout
      - setup_remote_docker:
          version: 17.03.0-ce
      - run:
          name: Build Base Image
          command: |
            set -x
            docker build -t baseimage -f docker/baseimage/Dockerfile .
      - run:
          name: Build Webapp Image
          command: |
            set -x
            docker build -t webapp -f docker/webapp/Dockerfile .
      - run:
          name: Build Celery Image
          command: |
            set -x
            docker build -t celery -f docker/celery/Dockerfile .
      - run:
          name: Build Stack Tester Image
          command: |
            set -x
            docker build -t stacktest -f docker/stacktest/Dockerfile .
      - run:
          name: Run Django Unit Tests
          command: |
            set -x
            docker-compose -f docker/docker-compose-unit-test.yaml run unit-test
            docker-compose -f docker/docker-compose-unit-test.yaml down
      - run:
          name: Run Integration Tests Against Local Docker-Compose Stack
          command: |
            set -x
            docker-compose -f docker/docker-compose-local-image-stack.yaml up -d
            docker run -e SERVICE="localhost:8001" --network container:stockpicker_webapp stacktest ./integration-tests.sh \
                || \
                (echo "*** WORKER1 LOGS:" && echo "$(docker logs stockpicker_worker1)" && \
                echo "*** WORKER2 LOGS:" && echo "$(docker logs stockpicker_worker2)" && \
                echo "*** WEBAPP LOGS:" && echo "$(docker logs stockpicker_webapp)" && \
                exit 1)
            docker-compose -f docker/docker-compose-local-image-stack.yaml down
      - run:
          name: Push Docker Images
          command: |
            set -x
            export PATH="$PATH:$(python -m site --user-base)/bin"
            $(aws ecr get-login --no-include-email --region us-east-2)
            IMAGE_TAG=$(echo "$CIRCLE_BRANCH" | sed 's/[^a-zA-Z0-9]/-/g'| sed -e 's/\(.*\)/\L\1/')
            AWS_REGION=us-east-2
            ECR_ID=421987441365
            docker tag webapp $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/webapp:$IMAGE_TAG
            docker tag celery $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/celery:$IMAGE_TAG
            docker push $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/webapp:$IMAGE_TAG
            docker push $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/celery:$IMAGE_TAG

  tagged_image_test:
    <<: *defaults
    steps:
      - checkout
      - setup_remote_docker:
          version: 17.03.0-ce
      - run:
          name: Build Stack Tester Image
          command: |
            set -x
            docker build -t stacktest -f docker/stacktest/Dockerfile .
      - run:
          name: Run Integration Tests With Pushed Images
          command: |
            set -x
            export IMAGE_TAG=$(echo "$CIRCLE_BRANCH" | sed 's/[^a-zA-Z0-9]/-/g'| sed -e 's/\(.*\)/\L\1/')
            export AWS_REGION=us-east-2
            export ECR_ID=421987441365
            $(aws ecr get-login --no-include-email --region us-east-2)
            docker-compose -f docker/docker-compose-tagged-image-stack.yaml up -d
            docker run -e SERVICE="localhost:8001" --network container:stockpicker_webapp stacktest ./integration-tests.sh \
                || \
                (echo "*** WORKER1 LOGS:" && echo "$(docker logs stockpicker_worker1)" && \
                echo "*** WORKER2 LOGS:" && echo "$(docker logs stockpicker_worker2)" && \
                echo "*** WEBAPP LOGS:" && echo "$(docker logs stockpicker_webapp)" && \
                exit 1)
            docker-compose -f docker/docker-compose-tagged-image-stack.yaml down

workflows:
  version: 2
  build-and-deploy:
    jobs:
      - image_build_test_push
      - tagged_image_test:
          requires:
            - image_build_test_push
```

We've added a "Push Docker Images" task to the original `image_build_test_push` CircleCI job, and added a second job as well. 
The new job is called `tagged_image_test`, and it will only execute if the `image_build_test_push` job completes.

The `tagged_image_test` relies on a new Docker-Compose file we need to create.

Create `source/docker/docker-compose-tagged-image-stack.yaml` with the contents:

```yaml
version: '3.4'
services:

  redis:
    image: redis
    container_name: stockpicker_redis
    env_file:
     - ../container_environments/test-stack.yaml

  rabbitmq:
    image: rabbitmq:3.6
    container_name: stockpicker_rabbitmq
    env_file:
     - ../container_environments/test-stack.yaml

  postgres:
    image: postgres:9.4
    container_name: stockpicker_postgres
    env_file:
     - ../container_environments/test-stack.yaml

  webapp:
    image: $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/webapp:$IMAGE_TAG
    container_name: stockpicker_webapp
    ports:
     - 8080:8001
    links:
      - postgres
      - rabbitmq
      - redis
    depends_on:
      - postgres
      - rabbitmq
      - redis
    env_file:
     - ../container_environments/test-stack.yaml

  worker1:
    image: $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/celery:$IMAGE_TAG
    container_name: stockpicker_worker1
    links:
      - postgres
      - rabbitmq
      - redis
    depends_on:
      - postgres
      - rabbitmq
      - redis
    env_file:
     - ../container_environments/test-stack.yaml
    command: bash -c "celery worker -O fair -c 1 --app=stockpicker.celery --loglevel=info"

  worker2:
    image: $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/celery:$IMAGE_TAG
    container_name: stockpicker_worker2
    links:
      - postgres
      - rabbitmq
      - redis
    depends_on:
      - postgres
      - rabbitmq
      - redis
    env_file:
     - ../container_environments/test-stack.yaml
    command: bash -c "celery worker -O fair -c 1 --app=stockpicker.celery --loglevel=info"

  beat:
    image: $ECR_ID.dkr.ecr.$AWS_REGION.amazonaws.com/stockpicker/celery:$IMAGE_TAG
    container_name: stockpicker_beat
    links:
      - postgres
      - rabbitmq
      - redis
    depends_on:
      - postgres
      - rabbitmq
      - redis
    env_file:
     - ../container_environments/test-stack.yaml
    command: bash -c "sleep 90 && celery beat --app=stockpicker.celery --loglevel=info"
 ```

We can now test the tagged, pushed images locally by running the following `bash` command:

```bash
docker-compose -f docker/docker-compose-tagged-image-stack.yaml up -d && \
docker run -e SERVICE="localhost:8001" --network container:stockpicker_webapp stacktest ./integration-tests.sh && \
docker-compose -f docker/docker-compose-tagged-image-stack.yaml down
```

You should see your tagged images being pulled (downloaded), and then see the integration tests run through as before.
The process will take some time to run.

### Push code, watch CircleCI

Now we can commit all our changes and push code to the repository to set off another CircleCI build.

So from your `source` directory, run:

```bash
git add -A
git commit -m "add a commit message"
git push origin master
```

Now go back to your CircleCI dashboard page and watch the latest two jobs run.

The final output you see in the second job, called `tagged_image_test` should look like the integration test output you have seen before.

[Prev: Part 3](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-3-ci-integration-testing.md)
|
[Next: Part 5](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-5-orchestration-kubernetes-rancher.md)
