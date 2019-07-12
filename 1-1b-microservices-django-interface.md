# Part 1-B: Microservices: A Modest Django Web App, Views and UI

In this part of the tutorial we will finish out the toy Stock-Picker application with some [Django views](https://docs.djangoproject.com/en/2.2/topics/http/views/), more [unit tests](https://docs.djangoproject.com/en/2.2/topics/testing/overview/), and an HTML/JavaScript UI.

### Complete Part 1A

Make sure you've completed the first part of the tutorial.

I'll assume that you still have all the files in place from the previous exercise, contained in your `source` directory at the top level of your local clone of the `devops-toolkit` [repostitory](https://github.com/sloanahrens/devops-toolkit).

Your `source` directory should have at least the following structure:

```
devops-toolkit/
    source/
        django/
            stockpicker/
                ...
            requirements.txt
```

I won't list all the files in the `stockpicker` directory, but if you have not properly completed [Part 1](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-1a-microservices-django-data.md) then what follows will probably not work for you.

### `devops` Development Environment

In this exercise we'll use the `devops` development environment (rather than the `baseimage` dev env from Part 2), so make sure you can run it as described [here](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/0-local-dev-env-devops.md).

Make sure you have built the `devops` Docker image, by running the following command from the `devops-toolkit` directory on your host OS:

```bash
docker build -t devops -f docker/devops/Dockerfile .
```

Go to your `source` directory now with:

```bash
cd source
```

Now start the development environment Docker container with:

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

Run `ls` from inside the container and you should see the files you created in the `source` directory:

```
root@1bc94b877039:/src# ls
django
```

If you see all the directories in the `devops-toolkit` directory, you're in the wrong place, and you need to go to your `source` directory one level down.

Now, since we have recycled our dev environment, we have to install our `pip` requirements again, so run:

```bash
pip install -r /src/django/requirements.txt
```

### pip install Django REST Framework

Now we're going to start adding Django Views.
Our API views will use the [Django REST Framework](https://www.django-rest-framework.org/), so we need to install it with:

```bash
pip install djangorestframework==3.9.3
pip freeze > /src/django/requirements.txt
```

### Django settings

We need to add a few settings in `settings.py`.
We will need to be sure that the `stockpicker` app itself has been added to `INSTALLED_APPS` in settings, so that the `staticfiles` app will find our static files.
We also need to add `'rest_framework'` to `INSTALLED_APPS`.

We need to make sure the following settings are in `source/django/stockpicker/stockpicker/settings.py`.
You can just paste this code at the bottom of the `source/django/stockpicker/stockpicker/settings.py` file, or replace the existing settings:

```python
STATICFILES_STORAGE = 'django.contrib.staticfiles.storage.StaticFilesStorage'
STATIC_URL = '/static/'
STATIC_ROOT = "/src/_static"
 
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'rest_framework',

    'stockpicker',
    'tickers',
]
```

### Building API views, with tests

To build our API views, accompanying URLs, and unit tests, we will need to edit four files.

**Step 1**: Create the following file.

`source/django/stockpicker/stockpicker/views.py`:
```python
from django.views.generic import TemplateView


class PickerPageView(TemplateView):

    template_name = 'picker.html'

```

**Step 2**:  We need to edit our [`urls.py` file](https://tutorial.djangogirls.org/en/django_urls/).

Edit `source/django/stockpicker/stockicker/urls.py` to match:
```python
from django.urls import path
from django.contrib import admin
from django.conf.urls.static import static
from django.conf import settings

from stockpicker.views import PickerPageView
from tickers.views import (TickersLoadedView,
                           SearchTickerDataView,
                           GetRecommendationsView,
                           AddTickerView)

urlpatterns = [
    path('admin/', admin.site.urls),

    path('', PickerPageView.as_view(), name='picker_page'),

    path('tickers/tickerlist/', TickersLoadedView.as_view(), name='tickers_loaded'),

    path('tickers/tickerdata/', SearchTickerDataView.as_view(), name='search_ticker_data'),

    path('tickers/recommendations/', GetRecommendationsView.as_view(), name='get_recommendations'),

    path('tickers/addticker/', AddTickerView.as_view(), name='add_ticker'),

] + static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)

```

**Step 3**:  Edit the contents of the `tickers/views` file.

`source/django/stockpicker/tickers/views.py` should match:
```python
from django.conf import settings

from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.status import HTTP_412_PRECONDITION_FAILED, HTTP_200_OK
from rest_framework.authentication import SessionAuthentication
from rest_framework.permissions import AllowAny

from tickers.models import Ticker, Quote
from tickers.utility import update_ticker_data


class SearchTickerDataView(APIView):
    authentication_classes = (SessionAuthentication,)
    permission_classes = (AllowAny,)

    def post(self, request, format=None):

        ticker_symbol = request.data['ticker']
        try:
            ticker = Ticker.objects.get(symbol=ticker_symbol)
        except Ticker.DoesNotExist:
            return Response({'success': False, 'error': 'Ticker "{0}" does not exist'.format(ticker_symbol)},
                            status=HTTP_412_PRECONDITION_FAILED)

        return Response({
            'success': True,
            'ticker': ticker.symbol,
            'index': settings.INDEX_TICKER,
            'avg_weeks': settings.MOVING_AVERAGE_WEEKS,
            'results': [quote.serialize() for quote in ticker.quote_set.order_by('date')]
        }, status=HTTP_200_OK)


class TickersLoadedView(APIView):
    authentication_classes = (SessionAuthentication,)
    permission_classes = (AllowAny,)

    def get(self, request, format=None):

        return Response({'tickers': [t.symbol for t in Ticker.objects.order_by('symbol')]}, status=HTTP_200_OK)


class GetRecommendationsView(APIView):
    authentication_classes = (SessionAuthentication,)
    permission_classes = (AllowAny,)

    def get(self, request, format=None):

        try:
            index_ticker = Ticker.objects.get(symbol=settings.INDEX_TICKER)
        except Ticker.DoesNotExist:
            return Response({'success': False, 'error': 'Ticker "{0}" does not exist'.format(settings.INDEX_TICKER)},
                            status=HTTP_412_PRECONDITION_FAILED)

        if index_ticker.latest_quote_date() is None:
            return Response({'success': False, 'error': 'No Quotes available.'}, status=HTTP_412_PRECONDITION_FAILED)

        sell_hits = Quote.objects.filter(date=index_ticker.latest_quote_date(),
                                         sac_to_sacma_ratio__gt=1).order_by('-sac_to_sacma_ratio')
        buy_hits = Quote.objects.filter(date=index_ticker.latest_quote_date(),
                                        sac_to_sacma_ratio__gt=0,
                                        sac_to_sacma_ratio__lt=1).order_by('sac_to_sacma_ratio')
        return Response({
            'success': True,
            'latest_data_date': index_ticker.latest_quote_date().strftime('%Y-%m-%d'),
            'sell_hits': [quote.serialize() for quote in list(sell_hits)[:25]],
            'buy_hits': [quote.serialize() for quote in list(buy_hits)[-25:]]
        }, status=HTTP_200_OK)


class AddTickerView(APIView):
    authentication_classes = (SessionAuthentication,)
    permission_classes = (AllowAny,)

    def post(self, request, format=None):

        ticker_symbol = request.data['ticker']
        update_ticker_data(ticker_symbol, force=True)

        return Response({'success': True}, status=HTTP_200_OK)

```

**Step 4**: Finally, we need to add unit tests for the views.

Edit the contents of `source/django/stockpicker/tickers/tests.py` to match:

```python
import json
import random

from django.utils.timezone import datetime
from django.test import TestCase
from django.urls import reverse
from django.conf import settings

from tickers.models import Ticker, Quote


def fake_quote_serialize(quote):
    return {
        'symbol': quote.ticker.symbol,
        'date': quote.date.strftime('%Y-%m-%d'),
        'ac': round(float(quote.adj_close), settings.DECIMAL_DIGITS),
        'iac': round(float(quote.index_adj_close), settings.DECIMAL_DIGITS),
        'sac': round(float(quote.scaled_adj_close), settings.DECIMAL_DIGITS),
        'sac_ma': round(float(quote.sac_moving_average), settings.DECIMAL_DIGITS),
        'ratio': round(float(quote.sac_to_sacma_ratio), settings.DECIMAL_DIGITS)
    }


class TickerModelTests(TestCase):

    def setUp(self):
        Ticker.objects.create(symbol='TEST')

    def test_ticker_exists(self):
        self.assertTrue(Ticker.objects.get(symbol='TEST').id > 0)


class QuoteModelTests(TestCase):

    def setUp(self):
        Quote.objects.create(ticker=Ticker.objects.create(symbol='TEST'), date=datetime.today())

    def test_quote_exists(self):
        ticker = Ticker.objects.get(symbol='TEST')
        self.assertTrue(Quote.objects.get(ticker=ticker, date=datetime.today()).id > 0)

    def test_quote_serializes(self):
        ticker = Ticker.objects.get(symbol='TEST')
        quote = Quote.objects.get(ticker=ticker, date=datetime.today())
        self.assertEqual(
            json.dumps(quote.serialize()),
            json.dumps(fake_quote_serialize(quote)))


class TickersLoadedViewTests(TestCase):

    def test_no_tickers(self):
        response = self.client.get(reverse('tickers_loaded'))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(
            json.dumps(response.data),
            json.dumps({'tickers': []}))

    def test_ticker_exists(self):
        Ticker.objects.create(symbol='TEST')
        response = self.client.get(reverse('tickers_loaded'))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(
            json.dumps(response.data),
            json.dumps({'tickers': ["TEST"]}))


class SearchTickerDataViewTests(TestCase):

    def test_no_ticker(self):
        response = self.client.post(reverse('search_ticker_data'),
                                    content_type="application/json",
                                    data=json.dumps({'ticker': 'NOPE'}))
        self.assertEqual(response.status_code, 412)
        self.assertEqual(
            json.dumps(response.data),
            json.dumps({'success': False,
                        'error': 'Ticker "NOPE" does not exist'}))

    def test_ticker_exists_but_no_data(self):
        Ticker.objects.create(symbol='TEST')
        response = self.client.post(reverse('search_ticker_data'),
                                    content_type="application/json",
                                    data=json.dumps({'ticker': 'TEST'}))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(
            json.dumps(response.data),
            json.dumps({'success': True,
                        'ticker': 'TEST',
                        'index': settings.INDEX_TICKER,
                        'avg_weeks': 86, 'results': []}))

    def test_ticker_has_one_quote(self):
        def rand_value():
            return random.uniform(0, 10)
        ticker = Ticker.objects.create(symbol='TEST')
        quote = Quote.objects.create(ticker=ticker, date=datetime.today())
        quote.adj_close = rand_value()
        quote.index_adj_close = rand_value()
        quote.scaled_adj_close = rand_value()
        quote.sac_moving_average = rand_value()
        quote.sac_to_sacma_ratio = rand_value()
        quote.save()
        response = self.client.post(reverse('search_ticker_data'),
                                    content_type="application/json",
                                    data=json.dumps({'ticker': 'TEST'}))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(
            json.dumps(response.data),
            json.dumps({'success': True,
                        'ticker': 'TEST',
                        'index': settings.INDEX_TICKER,
                        'avg_weeks': 86, 'results': [fake_quote_serialize(quote)]}))


class GetRecommendationsViewTests(TestCase):

    def test_index_ticker_does_not_exist(self):
        response = self.client.get(reverse('get_recommendations'))
        self.assertEqual(response.status_code, 412)
        self.assertEqual(
            json.dumps(response.data),
            json.dumps({'success': False,
                        'error': 'Ticker "{0}" does not exist'.format(settings.INDEX_TICKER)}))

    def test_no_quotes_available(self):
        Ticker.objects.create(symbol=settings.INDEX_TICKER)
        response = self.client.get(reverse('get_recommendations'))
        self.assertEqual(response.status_code, 412)
        self.assertEqual(
            json.dumps(response.data),
            json.dumps({'success': False,
                        'error': 'No Quotes available.'}))

    def test_buy_recommendation(self):
        Quote.objects.create(ticker=Ticker.objects.create(symbol=settings.INDEX_TICKER),
                             date=datetime.today())
        quote = Quote.objects.create(ticker=Ticker.objects.create(symbol='TEST'),
                                     date=datetime.today())
        quote.sac_to_sacma_ratio = 0.5
        quote.save()
        response = self.client.get(reverse('get_recommendations'))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(
            json.dumps(response.data),
            json.dumps({'success': True,
                        'latest_data_date': quote.date.strftime('%Y-%m-%d'),
                        'sell_hits': [],
                        'buy_hits': [fake_quote_serialize(quote)]}))

    def test_sell_recommendation(self):
        Quote.objects.create(ticker=Ticker.objects.create(symbol=settings.INDEX_TICKER),
                             date=datetime.today())
        quote = Quote.objects.create(ticker=Ticker.objects.create(symbol='TEST'),
                                     date=datetime.today())
        quote.sac_to_sacma_ratio = 1.5
        quote.save()
        response = self.client.get(reverse('get_recommendations'))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(
            json.dumps(response.data),
            json.dumps({'success': True,
                        'latest_data_date': quote.date.strftime('%Y-%m-%d'),
                        'sell_hits': [fake_quote_serialize(quote)],
                        'buy_hits': []}))

```

Now we can run the tests with `python manage.py test`, and the output should look like:

```
root@d100b5105a84:/src/django/stockpicker# python manage.py test
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
............
----------------------------------------------------------------------
Ran 12 tests in 0.305s

OK
Destroying test database for alias 'default'...
```

Notice that we ran 12 tests this time, instead of only the 3 test from Part 1-A.

### Front-end files

For our main app page we will need an HTML [Django Template](https://docs.djangoproject.com/en/2.2/topics/templates/).
So we'll need to create `picker.html` with the following contents.

`source/django/stockpicker/stockpicker/templates/picker.html` should match:
```html
{% load static %}
<!DOCTYPE html>
<html>
  <head>
    <title>Stock-Picker</title>

    <link rel="shortcut icon" type="image/png" href="{% static 'favicon.ico' %}"/>

    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap.min.css">
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap-theme.min.css">

    <link rel="stylesheet" type="text/css" href="{% static 'css/app.css' %}">

    <!--[if lt IE 9]>
    <script src="http://html5shim.googlecode.com/svn/trunk/html5.js"></script>
    <![endif]-->
  </head>
  <body>

    <div class="jumbotron">
      <div class="container">
          <h2>Toy Stock-Picker</h2>
          <p>
              This app uses an 86-week moving average of scaled daily price (adjusted closing price divided by SPY index value), as a decision boundary for <em>buy</em> versus <em>sell</em> classifications.</h4>
          </p>
          <div id="error_message" style="display:none;" class="alert alert-danger">add ticker error</div>
          <div id="success_message" style="display:none;" class="alert alert-success">add ticker success</div>
      </div>
    </div>

    <div class='container'>

        <div class="row">
          <div class="col-xs-2">
              <form id="add_ticker_form">
                <input type="text" style="width:80%" class="form-control" id="add_ticker_text" placeholder="TICKER">
                <button type="submit" id="add_ticker_btn" style="width:80%" class="btn btn-default">
                  Add Ticker
                </button>
              </form>
              <h4 style="margin-bottom: 5px;">Tickers Loaded</h4>
              <ul id="tickerlist"></ul>
          </div>
          <div class="col-xs-10">
              <h3 style="margin-top: 0;">Latest data date: <span id="latest_date"></span></h3>
              <div class="chart-cont">
                <div id="chart1stats" class="stats pull-right">*</div>
                <div id='chart1' class="chart"></div>
                <div class='marker' style="display:none;"></div>
                <div style="clear:both;"></div>
              </div>
              <button class="unzoom btn btn-default">
                Reset Zoom
              </button>
              <div class="chart-cont">
                <div id="chart2stats" class="stats pull-right">*</div>
                <div id='chart2' class="chart"></div>
                <div class='marker' style="display:none;"></div>
                <div style="clear:both;"></div>
              </div>
              <hr>
              <div>Filter: <span id="filter_rec"></span></div>
            <div class="row">
              <div class="col-xs-5">
                <h3>Buy (<span id="buy_count"></span>)</h3>
                <table class="table">
                  <thead>
                    <tr>
                      <th>sym (gf)</th>
                      <th>ac</th>
                      <th>sac</th>
                      <th>sacma</th>
                      <th>ratio</th>
                    </tr>
                  </thead>
                </table>
              </div>
              <div class="col-xs-5">
                <h3>Sell (<span id="sell_count"></span>)</h3>
                <table class="table">
                  <thead>
                    <tr>
                      <th>sym (gf)</th>
                      <th>ac</th>
                      <th>sac</th>
                      <th>sacma</th>
                      <th>ratio</th>
                    </tr>
                  </thead>
                </table>
              </div>
            </div>
            <div class="row" style="height: 500px; overflow:auto;">
              <div class="col-xs-5">
                <table class="table">
                  <tbody id="buy_table"></tbody>
                </table>
              </div>
              <div class="col-xs-5">
                <table class="table">
                  <tbody id="sell_table"></tbody>
                </table>
              </div>
            </div>
          </div>
        </div>

    </div>

    <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.11.3/jquery.min.js"></script>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/js/bootstrap.min.js"></script>

    <script type="text/javascript" src="{% static 'js/lib/jquery.flot.min.js' %}"></script>
    <script type="text/javascript" src="{% static 'js/lib/jquery.flot.selection.js' %}"></script>
    <script type="text/javascript" src="{% static 'js/picker.js' %}"></script>

  </body>
</html>
```

We also need several static files for the front-end, and now we're going to cheat a little bit, and just copy some code files over from the existing `devops-toolkit` source.
Feel free to explore these files as you like (sorry about the JavaScript; I know there are better ways, I just haven't gotten that far yet), but I'm not going to reproduce them here.

From your host OS, in the `devops-toolkit` directory, run the following copy command (or its equivalent):

```bash
cp -R django/stockpicker/stockpicker/static/ source/django/stockpicker/stockpicker/static/
```

*Note:* If your `source` directory is NOT in the root of your local copy of `devops-toolkit`, this command will fail.
I'll leave fixing it to you as homework, in that case.


You should now see a file structure that looks like:

```
stockpicker/
    stockpicker/
        static/
            css/
                app.css
            js/
                lib/
                    jquery.flot.min.js
                    jquery.flot.selection.js
                picker.js
            favicon.ico
        ...
    tickers/
    ...
```

### Port-forwarding from the development environment

Now we are ready to run the [Django development server](https://docs.djangoproject.com/en/2.2/intro/tutorial01/#the-development-server), but since we are running in a Docker container we will not be able to access the running application from the host OS.

To get around this, we need to modify our development environment command to forward the appropriate port.
If you are currently running the development environment, exit by typing `exit` and hitting enter.
As always, make sure you are in your `source` directory, and if your devops environment is running, `exit` out of it. 
Now run:

```bash
docker run -it \
  -p 8000:8000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD:/src \
  --name local_devops \
  --rm \
  devops \
  /bin/bash
```

Notice that we added `-p 8000:8000` to forward the port `8000`.

But now, since we recycled our dev environment, we have to install our `pip` requirements again, so run:

```bash
pip install -r /src/django/requirements.txt
```

Now run the [`collectstatic` management command](https://stackoverflow.com/questions/34586114/whats-the-point-of-djangos-collectstatic):

```bash
cd /src/django/stockpicker && python manage.py collectstatic
```

You should see:

```
root@89668020f81d:/src/django# cd /src/django/stockpicker && python manage.py collectstatic

157 static files copied to '/src/_static'.
root@89668020f81d:/src/django/stockpicker#
```

### Run the Django development server

Now, we can finally run our development server, with:

```bash
cd /src/django/stockpicker
python manage.py runserver 0.0.0.0:8000
```

You should see the webserver running in the log output. 
Leave it running and open your web browser.

Now you should be able to go to [`localhost:8000`](http://localhost:8000/) in your [favorite web-browser](https://www.mozilla.org) from your host OS, and see the same thing that you see live at [stockpicker.sloanahrens.com](https://stockpicker.sloanahrens.com).

You can stop the development server with `ctl-c` and exit your running `devops` container by typing `exit` and hitting enter.

In the next exercise we will containerize the application, and introduce a slightly different development environment.

[Prev: Part 1A](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-1a-microservices-django-data.md)
|
[Next: Part 2](https://github.com/sloanahrens/devops-toolkit-tutorials/blob/master/1-2-containerization-celery.md)
