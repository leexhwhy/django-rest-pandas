Django REST Pandas
==================

#### [Django REST Framework] + [Pandas] = A Model-driven Visualization API

**Django REST Pandas** (DRP) provides a simple way to generate and serve [Pandas] DataFrames via the [Django REST Framework].  The resulting API can serve up CSV (and a number of [other formats](#formats)) for consumption by a client-side visualization tool like [d3.js].  The actual client implementation is left to the user - giving full flexibility for whatever visualizations you want to come up with.  (That said, if you want some out of the box d3-powered charts for use with DRP, you may be interested in [wq.app]'s [chart.js] and/or [wq.db]'s [chart] module.)

[![Build Status](https://travis-ci.org/wq/django-rest-pandas.png?branch=master)](https://travis-ci.org/wq/django-rest-pandas) 

## Related Work
The field of Python-powered data analysis and visualization is growing, and there are a number of similar solutions that may fit your needs better.

 * [Django Pandas] provides a custom model manager with Pandas support.  By contrast, Django REST Pandas works at the view level, by adding Pandas support via a Django REST Framework serializer.
 * [DRF-CSV] provides CSV renderers for use with Django REST Framework.  It may be useful if you just want a CSV API and don't have a need for the Pandas DataFrame functionality.
 * [Bokeh] is a complete client-server visualization platform.  It does not leverage d3 or Django, but is notable as a ground-up approach to addressing similar use cases.
 * [mpld3] provides a direct bridge from [matplotlib] to [d3.js], complete with seamless [IPython] integration.  It is "limited" to matplotlib charts but should be sufficient for many use cases.

The goal of Django REST Pandas is to provide a generic REST API for serving up dataframes.  In this sense, it is similar to the Plot Server in Bokeh, but more generic in that it does not assume any particular client technology (which can be good or bad depending on your use case).  Further, DRP is optimized for integration with public-facing Django-powered websites (unlike mpld3 which is primarily intended for use within IPython.)

## Usage

### Getting Started

```bash
pip install rest-pandas
```

### Usage Example

```python
# views.py
from rest_pandas import PandasView
from .models import TimeSeries
class TimeSeriesView(PandasView):
    model = TimeSeries
    
    # In response to get(), the underlying Django REST Framework ListAPIView
    # will load the default queryset (self.model.objects.all()) and then pass
    # it to the following function.
    
    def filter_queryset(self, qs): 
        # At this point, you can filter queryset based on self.request or other
        # settings (useful for limiting memory usage)
        return qs
        
    # Then, the included PandasSerializer will serialize the queryset into a
    # simple list of dicts (using the DRF ModelSerializer).  To customize
    # which fields to include, subclass PandasSerializer and set the
    # appropriate ModelSerializer options.  Then, set the serializer_class
    # property on the view to your PandasSerializer subclass.
    
    # Next, the PandasSerializer will load the ModelSerializer result into a
    # DataFrame and pass it to the following function on the view.

    def transform_dataframe(self, dataframe):
        # Here you can transform the dataframe based on self.request
        # (useful for pivoting or computing statistics)
        return dataframe
        
    # Finally, the included Renderers will process the dataframe into one of
    # the output formats below.
```

```python
# urls.py
from django.conf.urls import patterns, include, url
from rest_framework.urlpatterns import format_suffix_patterns

from .views import TimeSeriesView
urlpatterns = patterns('',
    url(r'^data', TimeSeriesView.as_view()),
)
urlpatterns = format_suffix_patterns(urlpatterns)
```

The default `PandasView` will serve up all of the available data from the provided model in a simple tabular form.  You can also use a `PandasViewSet` if you are using Django REST Framework's ViewSets and Routers, or a `PandasSimpleView` if you would just like to serve up some data without a Django model as the source.

## Implementation
The underlying implementation is a set of [serializers] that take the normal serializer result and put it into a dataframe.  Then, the included [renderers] generate the output using the built in Pandas functionality.  

### Formats

The following output formats are provided by default.  These are provided as renderer classes in order to leverage the content negotiation built into Django REST Framework.  This means clients can specify a format via `Accepts: text/csv` or by appending `.csv` to the URL (if the above `urls.py` is followed).

Format | Content Type | Pandas Dataframe Function | Notes
-------|--------------|---------------------------|--------
CSV    | `text/csv` | `to_csv()` |
TXT    | `text/plain` | `to_csv()` | Useful for testing, as most browsers will download a CSV file instead of displaying it
JSON   | `application/json` | `to_json()` |
XLSX   | `application/vnd.openxml...sheet` | `to_excel()` |
XLS    | `application/vnd.ms-excel` | `to_excel()` 
PNG    | `image/png` | `plot()` | Currently not very customizable, but a simple way to view the data as an image. 
SVG    | `image/svg` | `plot()` | Eventually these could become a fallback for clients that can't handle d3.js

Perhaps counterintuitively, the CSV renderer is the default in Django REST Pandas, as it is the most stable and useful for API building.  While the Pandas JSON serializer is improving, the primary reason for making CSV the default is the compactness it provides over JSON when serializing time series data.  This is particularly valuable for Pandas dataframes, in which:

 - each record has the same keys, and
 - there are (usually) no nested objects

While a normal CSV file only has a single row of column headers, Pandas can produce files with nested columns.  This is a useful way to provide metadata about time series that is difficult to represent in a plain CSV file.  However, it also makes the resulting CSV more difficult to parse.  For this reason, you may be interested in [wq/pandas.js], a d3 extension for loading the complex CSV generated by Pandas Dataframes.

```javascript
// mychart.js
define(['d3', 'wq/pandas'], function(d3, pandas) {

d3.csv("/data.csv", render);
// Or
pandas.get('/data.csv' render);

function render(error, data) {
    d3.select('svg')
       .selectAll('rect')
       .data(data)
       // ...
}

});

```

You can override the default renderers by setting `PANDAS_RENDERERS` in your `settings.py`, or by overriding `renderer_classes` in your `PandasView` subclass.

[Django REST Framework]: http://django-rest-framework.org
[Pandas]: http://pandas.pydata.org
[d3.js]: http://d3js.org
[wq.app]: http://wq.io/wq.app
[chart.js]: http://wq.io/docs/chart-js
[wq.db]: http://wq.io/wq.db
[chart]: http://wq.io/docs/chart
[Django Pandas]: https://github.com/chrisdev/django-pandas/
[bokeh]: http://bokeh.pydata.org/
[mpld3]: https://github.com/jakevdp/mpld3
[DRF-CSV]: https://github.com/mjumbewu/django-rest-framework-csv
[matplotlib]: http://matplotlib.org/
[IPython]: http://ipython.org/
[serializers]: https://github.com/wq/django-rest-pandas/blob/master/rest_pandas/serializers.py
[renderers]: https://github.com/wq/django-rest-pandas/blob/master/rest_pandas/renderers.py
[wq/pandas.js]: http://wq.io/docs/pandas-js
