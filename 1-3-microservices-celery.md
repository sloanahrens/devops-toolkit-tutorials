# Part 3: Async Task Processing with Celery

In this exercise we'll take the Dockerized Django application we built in Parts 1 and 2, and add asynchronous task-processing capabilities with Celery.

### Complete Parts 1 and 2

Make sure you have completed [Part 1](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-1-microservices-django.md) and [Part 2](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-2-containerization-baseimage.md) of the tutorials already.

I'll assume that you have all the other files from Parts 1 and 2 still in place in the `devops-toolkit/source/django` directory.
This exercise should hopefully be [idempotent](https://en.wikipedia.org/wiki/Idempotence), so if you already have the other files in place too, it should still work.

### Development environment

We will use a development environment similar to that of Part 2, so build the `baseimage` image with:

```bash
docker build -t baseimage -f docker/baseimage/Dockerfile .
```

Now run the `baseimage` development environment with:

```bash
docker-compose -f docker/docker-compose-local-dev-django.yaml up
```

Lastly, make sure you can open another terminal tab and "exec" into the container with:

```bash
docker exec -it local_dev_baseimage /bin/bash
```

We need to add some stuff to our Python requirements, so this is a good time to do that. 
From the `docker exec` session you started with the last command, run:

```bash
pip install celery==4.3.0
pip install redis==3.2.1
pip install django-redis-cache==2.0.0
pip freeze > /src/django/requirements.txt
```

Exit the container by typing `exit`.

Assuming all that worked, now stop the stack by hitting `ctl-c` in the first tab, then run:

```bash
docker-compose -f docker/docker-compose-local-dev-django.yaml down
```

Now we need to rebuild the `baseimage` with the new requirements file:

```bash
docker build -t baseimage -f docker/baseimage/Dockerfile .
```

### Celery Django settings

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

Similar to the approach in [Part 2](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-2-containerization-baseimage.md), we'll use a Docker-Compose file for local development.

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

To add a Celery [beat scheduler](http://docs.celeryproject.org/en/latest/userguide/periodic-tasks.html), create the following file:

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

Now restart the environment again, and shortly you should see the data update task executing in the log output, and see working graphs at [http://localhost:8000](http://localhost.8000).

[Prev: Part 2](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-2-containerization-baseimage.md)
[Next: Part 4](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-4-ci-integration-testing.md)