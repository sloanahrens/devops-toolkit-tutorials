# Part 2: Dockerize the App

In this exercise we will take the Django application we built in Part 1, and "dockerize" it.
We will build the configuration necessary to build a Docker image that can run our application for local development purposes.
In later exercises we will build on this foundation to create production-ready application images via a continuous integration pipeline.

### Install Docker-Compose

If you don't have [Docker-Compose](https://docs.docker.com/compose/) installed on your host OS yet, you will need to install it now.
You can find installers for the various systems [here](https://docs.docker.com/compose/install/).

### Complete Part 1

Make sure you have completed [Part 1](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-1-microservices-django.md) of the tutorials already.

I'll assume that you have all the other files from Part 1 still in place in the `devops-toolkit/source/django` directory.
This tutorial should hopefully be [idempotent](https://en.wikipedia.org/wiki/Idempotence), so if you already have the other files in place too, it should still work.

### Development environment

We're not going to use the [`devops` development environment](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/0-local-dev-env-devops.md) in this tutorial like we did in Part 1.
Using the base Docker image for our application turns out to be a slightly better way, and hopefully less confusing in the end, since we will be using one of our deliverables as a development tool.

### PostgreSQL

In this exercise we are going to run multiple microservices, including PostgreSQL.
We made it through Part 1 using only Sqlite, but since we will be need to run multiple microservices in order to use Celery in [Part 3](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-3-microservices-celery.md), we may as well upgrade our database along the way.


### Add Postgres client to requirements file

We will need to add the [psycopg2 adapter](https://pypi.org/project/psycopg2/) to our `pip` requirements, if needed.

Make sure the `django/requirements file` in your `source` directory contains the following line:

```
psycopg2==2.8.2
```

So your `django/requirements.txt` file, up to this point, should contain at least the following (more is (probably) okay):

```
awscli==1.16.193
botocore==1.12.183
certifi==2019.6.16
chardet==3.0.4
colorama==0.3.9
Django==2.2.3
djangorestframework==3.9.3
docutils==0.14
fix-yahoo-finance==0.1.33
idna==2.7
jmespath==0.9.4
lxml==4.3.4
multitasking==0.0.9
numpy==1.16.4
pandas==0.24.2
pandas-datareader==0.7.0
psycopg2==2.8.2
pyasn1==0.4.5
python-dateutil==2.8.0
pytz==2019.1
PyYAML==5.1
requests==2.19.1
rsa==3.4.2
s3transfer==0.2.1
six==1.12.0
sqlparse==0.3.0
urllib3==1.23
wrapt==1.11.2

```

### Django database setting

We also need to add our database settings to `settings.py`:

So `source/django/stockpicker/stockpicker/settings.py` should include:
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': os.getenv('POSTGRES_DB', 'local_db'),
        'USER': os.getenv('POSTGRES_USER', 'local_user'),
        'PASSWORD': os.getenv('POSTGRES_PASSWORD', 'postgres-password'),
        'HOST': os.getenv('POSTGRES_HOST', 'localhost'),
        'PORT': os.getenv('POSTGRES_PORT', 5432),
    }
}
```

### 1) Docker base image

We're going to need to create several files now. 
Our first `Dockerfile` is next.

Make sure that your `source` directory exists at the root of the `devops-toolkit` directory, and create:

`source/docker/baseimage/Dockerfile`:

```dockerfile
FROM python:3.6-stretch

ENV PYTHONUNBUFFERED 1

WORKDIR /src

RUN apt-get update && apt-get install -y postgresql

COPY ./docker/scripts/wait-for-it.sh /usr/bin/wait-for-it.sh
RUN chmod 755 /usr/bin/wait-for-it.sh

COPY ./django/requirements.txt /src/requirements.txt

RUN pip --no-cache-dir install --progress-bar pretty -r requirements.txt
```

In this Dockerfile we start with the [`python:3.6-stretch` image](https://pythonspeed.com/articles/base-image-python-docker-images/), and install the [`apt`](https://wiki.debian.org/Apt) package `postgresql`.

[Wait-For-It](https://github.com/vishnubob/wait-for-it) is a handy script we will install below.

The last thing the `baseimage` Dockerfile does is install all the Python requirements in `requirements.txt`.
The `baseimage` can be cached, keyed on a hash of the `requirements.txt` file (see [this SO answer](https://stackoverflow.com/a/34399661/2551686)).
Docker itself does this, and we can also do it in CircleCI to slightly reduce build times.

### 2) Docker-Compose local dev file

Create: 

`source/docker/docker-compose-local-dev-django.yaml`:

```yaml
version: '3.4'

services:

  django:
    image: baseimage
    container_name: local_dev_baseimage
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_PORT: 5432
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: domicile-comanche-audible-bighorn
      POSTGRES_DB: db
    volumes:
      - ../:/src
    working_dir: /src/django/stockpicker
    ports:
      - "8000:8000"
    links:
      - postgres
    command: bash -c "wait-for-it.sh -t 60 postgres:5432 && python manage.py migrate && python manage.py runserver 0.0.0.0:8000"

  postgres:
    image: postgres:9.4
    container_name: local_dev_postgres
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_PORT: 5432
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: domicile-comanche-audible-bighorn
      POSTGRES_DB: db
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
    external: false
```

There's a lot going on in this file.
We're going to first build the `baseimage` image, and then we're going to use it via this Docker-Compose file.

The Docker-Compose file specifies two containers to run: an instance of our `baseimage` and a supporting `postgres` container.

We've provided some environment variables to both containers, and in the `baseimage` container we've connected the current working directory to the `/src` directory in the container, and forwarded port 8000 so we can view the running web app from the host OS.

The working directory in the `baseimage` container is connected to the directory holding our Django `manage.py` file, and we've specified a starting command that will run the data migrations (if needed) and then the Django development server.

The [postgres container](https://linuxhint.com/run_postgresql_docker_compose/) will run alongside our `baseimage` dev environment, attached to a [docker volume](https://docs.docker.com/storage/volumes/) so we will have data persistence.

### 3) wait-for-it.sh

[Wait-For-It](https://github.com/vishnubob/wait-for-it) is super handy in the world of containers.
Create: 
`source/docker/scripts/wait-for-it.sh` with the contents from: [https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh](https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh)

### Build `baseimage`

With all those files in place we can now build our first (well, our second) Docker image.

From your host OS, run:

```bash
docker build -t baseimage -f docker/baseimage/Dockerfile .
```

So this time, instead of installing all the Python requirements in the development environment, we are creating a `baseimage` for our application and installing the requirements in that Docker image.
The build will take a few minutes to finish.
After it's done you can see that it was built with `docker image ls`; you should see something along the lines of:

```
root@7b665110d575:/src# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
baseimage           latest              6db32e4ed244        32 seconds ago      1.12 GB
devops              latest              a9f536aaea32        10 hours ago        1.37 GB
python              3.6-stretch         48c06762acf0        8 days ago          924 MB
```

### Run local development "stack"

Now we can run the image, with environment variables and a PostgreSQL database using our Docker-Compose file.
From your local `source` directory, on your host OS, run:

```bash
docker-compose -f docker/docker-compose-local-dev-django.yaml up
```

Now you should be able to access the running app in your browswer at [http://localhost:8000/](http://localhost:8000/).

You can watch the log output for the two containers in the terminal tab from which you ran the `docker-compose up` command.
(You can add `-d` to the command to run in detached mode.)

You can "exec" into (analogous to "SSHing into") the running `baseimage` container, from another terminal tab/session, with:

```bash
docker exec -it local_dev_baseimage /bin/bash
```

As you change code from your host OS--using whatever text editor you want--the django development server will refresh so you can see the changes in the browser.

If you need to recycle the "stack" (consisting of two running containers) for whatever reason, you can stop it with `ctl-c`, and then run:

```bash
docker-compose -f docker/docker-compose-local-dev-django.yaml down
```

Then re-run this to start it again:

```bash
docker-compose -f docker/docker-compose-local-dev-django.yaml up
```

After you're done with it, be sure to kill the stack with:

```bash
docker-compose -f docker/docker-compose-local-dev-django.yaml down
```

[Prev: Part 1](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-1-microservices-django.md)
[Next: Part 3](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-3-microservices-celery.md)
