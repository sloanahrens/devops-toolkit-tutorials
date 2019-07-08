# Part 2: Containerization: Dockerize the Django app and add Celery

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


### Add dependencies to requirements file

We will need to add the [psycopg2 adapter](https://pypi.org/project/psycopg2/) to our `pip` requirements, as well as a few other Python dependencies.
The dependencies are [Celery](), [redis](), and [django-redis-cache]().
This time we will simply edit the `source/django/requirements.txt` file.

`source/django/requirements.txt` sound match:
```
amqp==2.5.0
awscli==1.16.193
billiard==3.6.0.0
botocore==1.12.183
celery==4.3.0
certifi==2019.6.16
chardet==3.0.4
colorama==0.3.9
Django==2.2.3
django-redis-cache==2.0.0
djangorestframework==3.9.3
docutils==0.14
fix-yahoo-finance==0.1.33
idna==2.7
jmespath==0.9.4
kombu==4.6.3
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
redis==3.2.1
requests==2.19.1
rsa==3.4.2
s3transfer==0.2.1
six==1.12.0
sqlparse==0.3.0
urllib3==1.23
vine==1.3.0
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

From your host OS, in the `devops-toolkit` directory, run:

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

### Celery Django settings

Now that we have our `baseimage` running with its `postgres` dependency via Docker-Compose, let's add Celery to the mix.

We will need a handful of Celery-related Django settings, so add the following to `settings.py`.

So `source/django/stockpicker/stockpicker/settings.py` should include:

```python
from celery.schedules import crontab

CELERY_BROKER_URL = 'amqp://{rabbitmq_user}:{rabbitmq_password}@{rabbitmq_host}:{rabbitmq_port}/{rabbitmq_namespace}'.format(
    rabbitmq_user=os.getenv('RABBITMQ_DEFAULT_USER', 'local_user'),
    rabbitmq_password=os.getenv('RABBITMQ_DEFAULT_PASS', 'rabbitmq_password'),
    rabbitmq_host=os.getenv('RABBITMQ_HOST', 'localhost'),
    rabbitmq_port=os.getenv('RABBITMQ_PORT', 5672),
    rabbitmq_namespace=os.getenv('RABBITMQ_DEFAULT_VHOST', 'local_vhost')
)

CELERY_RESULT_BACKEND = 'redis://{redis_host}:{redis_port}/{redis_namespace}'.format(
    redis_host=os.getenv('REDIS_HOST', 'localhost'),
    redis_port=os.getenv('REDIS_PORT', 6379),
    redis_namespace=0
)

CELERY_ALWAYS_EAGER = False
CELERYD_PREFETCH_MULTIPLIER = 1
CELERY_ACKS_LATE = True
CELERYD_MAX_TASKS_PER_CHILD = 1

CELERY_TIMEZONE = 'America/Chicago'

CELERY_BEAT_SCHEDULE = {
    'quotes-hourly-update': {
        'task': 'tickers.tasks.update_all_tickers',
        'schedule': crontab(hour="*/3", minute=0, day_of_week='mon,tue,wed,thu,fri'),
        # for local development testing:
        # 'schedule': crontab(hour="*", minute="*", day_of_week='*'),
    }
}

# redis cache
CACHES = {
    'default': {
        'BACKEND': 'redis_cache.RedisCache',
        'LOCATION': 'redis://{redis_host}:{redis_port}/{redis_namespace}'.format(
            redis_host=os.getenv('REDIS_HOST', 'localhost'),
            redis_port=os.getenv('REDIS_PORT', 6379),
            redis_namespace=1)
    },
}

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

Now we're going to need to edit a file and create four new files.

### 1) App init file

Edit the contents of  to match:

We're going to need to edit the contents of a Python `__init__.py` file now.

`source/django/stockpicker/stockpicker/__init__.py` should match:
```python
from __future__ import absolute_import, unicode_literals

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app

__all__ = ('celery_app',)

```

### 2) Celery app file

Create:

`source/django/stockpicker/stockpicker/celery.py` should match:

```python
import os
from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'stockpicker.settings')

app = Celery('stockpicker')

# Using a string here means the worker doesn't have to serialize
# the configuration object to child processes.
# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
app.autodiscover_tasks()


@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```

### 3) Celery tasks file

Now we need some Celery tasks.
Create: 

`source/django/stockpicker/tickers/tasks.py` should match:

```python
from django.conf import settings
from django.core.cache import cache

from stockpicker.celery import app
from tickers.models import Ticker
from tickers.utility import update_ticker_data


@app.task()
def update_all_tickers():

    update_ticker(settings.INDEX_TICKER)

    ticker_list = Ticker.objects.all().order_by('symbol')

    for ticker in ticker_list:
        update_ticker.delay(ticker.symbol)


@app.task()
def update_ticker(ticker_symbol):

    # cache.add returns False if the key already exists
    if not cache.add(ticker_symbol, 'true', 5 * 60):
        print('{0} has already been accepted by another task.'.format(ticker_symbol))
        return

    update_ticker_data(ticker_symbol)

    cache.delete(ticker_symbol)

    return ticker_symbol

```

This Celery task simply checks each Ticker iteratively (well, recursively) to make sure they are all updated.
This approach is a slight improvement over a more traditional loop through the `Ticker`s for reasons that won't come up until a later tutorial.

### 4) Container environment file

When deploying docker images we will need to be able to easily configure a number of environment variables inside the containers.
A straightforward way to do that is with [env files](https://docs.docker.com/compose/env-file/).
Create:

`source/container_environments/test-stack.yaml`:

```yaml
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_USER=test_user
POSTGRES_PASSWORD=domicile-comanche-audible-bighorn
POSTGRES_DB=db
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_DEFAULT_USER=test_user
RABBITMQ_DEFAULT_PASS=rabbit_password
RABBITMQ_DEFAULT_VHOST=test_vhost
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_NAMESPACE=0
SUPERUSER_PASSWORD=column-hand-pith-baby
SUPERUSER_EMAIL=admin@nowhere.com
APP_DEBUG=True
```

### 5) Docker-Compose celery stack file

Similar to the approach in [Part 2](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-2-containerization-celery.md), we'll use a Docker-Compose file for local development.

Create:

`source/docker/docker-compose-local-dev-django-celery.yaml`:

```yaml
version: '3.4'

services:

  django:
    image: baseimage
    container_name: local_dev_django
    env_file:
     - ../container_environments/test-stack.yaml
    volumes:
      - ../:/src
    working_dir: /src/django/stockpicker
    ports:
      - "8000:8000"
    links:
      - postgres
      - rabbitmq
      - redis
    command: bash -c "wait-for-it.sh -t 60 postgres:5432 && python manage.py migrate && python manage.py load_tickers && python manage.py runserver 0.0.0.0:8000"

  celery:
    image: baseimage
    container_name: local_dev_celery
    env_file:
     - ../container_environments/test-stack.yaml
    environment:
      C_FORCE_ROOT: "yes"
    working_dir: /src/django/stockpicker
    volumes:
      - ../:/src
    links:
      - postgres
      - rabbitmq
      - redis
    command: bash -c "celery worker -O fair -c 1 --app=stockpicker.celery --loglevel=info"

  redis:
    image: redis
    container_name: local_dev_redis
    env_file:
     - ../container_environments/test-stack.yaml

  rabbitmq:
    image: rabbitmq:3.6
    container_name: local_dev_rabbitmq
    env_file:
     - ../container_environments/test-stack.yaml

  postgres:
    image: postgres:9.4
    container_name: local_dev_postgres
    env_file:
     - ../container_environments/test-stack.yaml
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
    external: false
```

This Docker-Compose file creates the same `postgres` and `django` instances as in Part 1, and adds [redis](https://redis.io/) and [RabbitMQ](https://www.rabbitmq.com/) services as well as a second `baseimage` instance configured to run a single Celery worker. 

Now, with everything else in place, we can run it with:

```bash
docker-compose -f docker/docker-compose-local-dev-django-celery.yaml up
```

From another terminal tab run `docker ps` and you should see something like:

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                NAMES
a68fe47487e9        baseimage           "bash -c 'celery w..."   31 seconds ago      Up 29 seconds                                            local_dev_celery
aa9005f7fc7f        baseimage           "bash -c 'wait-for..."   31 seconds ago      Up 29 seconds       0.0.0.0:8000->8000/tcp               local_dev_django
d2a112daa25a        rabbitmq:3.6        "docker-entrypoint..."   32 seconds ago      Up 30 seconds       4369/tcp, 5671-5672/tcp, 25672/tcp   local_dev_rabbitmq
c5a4be7d1d44        redis               "docker-entrypoint..."   32 seconds ago      Up 30 seconds       6379/tcp                             local_dev_redis
71c55b24d437        postgres:9.4        "docker-entrypoint..."   32 seconds ago      Up 30 seconds       5432/tcp                             local_dev_postgres
```

Let's fire off the data update task by `exec`ing into the `local_dev_django` container and running some Python code.

Run:

```bash
docker exec -it local_dev_django /bin/bash
```

Then from inside the container run:

```bash
echo "from tickers.tasks import update_all_tickers; update_all_tickers.delay()" | python manage.py shell
```

Now watch the output in the tab running the Docker-Compose stack (or look at logs with `docker logs`) and you should see output similar to what you saw in Part 1 when updating stock quote data.

If you don't, it means your data is already up-to-date.
You can wipe your volume and start over by exiting and removing the current stack (`docker-compose down`), and then running:

```bash
docker volume rm docker_postgres_data
```

Now re-start the development stack, `exec` into the `local_dev_django` container, and run the above Python commands again, and you should see data-building task output.

### Celery Beat

To add a Celery [beat scheduler](http://docs.celeryproject.org/en/latest/userguide/periodic-tasks.html) that will update the stock quotes data automatically, create the following file:

`source/docker/docker-compose-local-dev-django-celery-beat.yaml`:

```yaml
version: '3.4'

services:

  django:
    image: baseimage
    container_name: local_dev_django
    env_file:
     - ../container_environments/test-stack.yaml
    volumes:
      - ../:/src
    working_dir: /src/django/stockpicker
    ports:
      - "8000:8000"
    links:
      - postgres
      - rabbitmq
      - redis
    command: bash -c "wait-for-it.sh -t 60 postgres:5432 && python manage.py migrate && python manage.py load_tickers && python manage.py runserver 0.0.0.0:8000"

  celery:
    image: baseimage
    container_name: local_dev_celery_worker
    env_file:
     - ../container_environments/test-stack.yaml
    environment:
      C_FORCE_ROOT: "yes"
    working_dir: /src/django/stockpicker
    volumes:
      - ../:/src
    links:
      - postgres
      - rabbitmq
      - redis
    command: bash -c "celery worker -O fair -c 1 --app=stockpicker.celery --loglevel=info"

  beat:
    image: baseimage
    container_name: local_dev_celery_beat
    env_file:
     - ../container_environments/test-stack.yaml
    environment:
      C_FORCE_ROOT: "yes"
    working_dir: /src/django/stockpicker
    volumes:
      - ../:/src
    links:
      - postgres
      - rabbitmq
      - redis
    command: bash -c "sleep 30 && celery beat --app=stockpicker.celery --loglevel=info"

  redis:
    image: redis
    container_name: local_dev_redis
    env_file:
     - ../container_environments/test-stack.yaml

  rabbitmq:
    image: rabbitmq:3.6
    container_name: local_dev_rabbitmq
    env_file:
     - ../container_environments/test-stack.yaml

  postgres:
    image: postgres:9.4
    container_name: local_dev_postgres
    env_file:
     - ../container_environments/test-stack.yaml
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
    external: false
```

This is the same as `docker-compose-local-dev-django-celery.yaml` with the addition of the Celery beat scheduler container.

The default Celery schedule runs the stock quote data update task once every three hours on buisiness days.
If you want to make it run more-or-less immediately, you can update `CELERY_BEAT_SCHEDULE` in `source/django/stockpicker/stockpicker/settings.py` to match:

```python
CELERY_BEAT_SCHEDULE = {
    'quotes-hourly-update': {
        'task': 'tickers.tasks.chained_ticker_updates',
        'schedule': crontab(hour="*", minute="*", day_of_week='*'),
    }
}
```

Then recycle the dev environment with:

```bash
docker-compose -f docker/docker-compose-local-dev-django-celery-beat.yaml down
docker-compose -f docker/docker-compose-local-dev-django-celery-beat.yaml up
```

If you see an error like this in the output:

```
local_dev_celery_beat | ERROR: Pidfile (celerybeat.pid) already exists.
local_dev_celery_beat | Seems we're already running? (pid: 1)
```

Then kill the dev environment, and remove the `celerybeat.pid` file with:

```bash
rm django/stockpicker/celerybeat.pid
```

Now restart the environment again, and shortly you should see the data update task executing in the log output, and eventually see working graphs at [http://localhost:8000](http://localhost.8000).

[Prev: Part 1](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-1-microservices-django.md)
|
[Next: Part 3](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-3-ci-integration-testing.md)



# devops-toolkit-tutorials
Tutorials describing construction of the devops-toolkit repository.

Start at the beginning: 

[0-local-dev-env-devops.md](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/0-local-dev-env-devops.md)