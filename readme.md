Dashboard real-time
Paquetes a instalar.

conda install datashader holoviews hvplot notebook numpy pandas panel param requests streamz

import operator as op
import numpy as np
import pandas as pd
import requests
import param
import panel as pn
import hvplot.pandas
import hvplot.streamz
import holoviews as hv
from holoviews.element.tiles import EsriImagery
from holoviews.selection import link_selections
from datashader.utils import lnglat_to_meters
from streamz.dataframe import PeriodicDataFrame


# Funcion para obtener datos meteolorologicos
def weather_data(cities, openweathermap_api_key=openweathermap_api_key):
    """
    Get weather data for a list of cities using the openweathermap API
    """
    L = []
    for c in cities:
        res = requests.get(f'http://api.openweathermap.org/data/2.5/weather?q={c}&appid={openweathermap_api_key}&units=imperial')
        L.append(res.json())

    df = pd.DataFrame(L)
    df['lon'] = df['coord'].map(op.itemgetter('lon'))
    df['lat'] = df['coord'].map(op.itemgetter('lat'))
    df['Temprature'] = df['main'].map(op.itemgetter('temp'))
    df['Humidity'] = df['main'].map(op.itemgetter('humidity'))
    df['Wind Speed'] = df['wind'].map(op.itemgetter('speed'))
    return df[['name','lon', 'lat','Temprature','Humidity','Wind Speed']]
    
#   
def streaming_weather_data(**kwargs):
    """
    callback function 
    get San Francisco weather data 
    """
    df = weather_data(['San Francisco'])
    df['time'] = [pd.Timestamp.now()]
    return df.set_index('time')

# Make a streaming dataframe 
df = PeriodicDataFrame(streaming_weather_data, interval='30s')

# panel dashboard for streaming data 
pn_realtime = pn.Column(
    pn.Row(
      df[['Temprature']].hvplot.line(title='Temprature', backlog=1000),
      df[['Humidity']].hvplot.line(title='Humidity', backlog=1000)),
  df[['Wind Speed']].hvplot.line(title='Wind Speed', backlog=1000)
)

cities = ['San Francisco', 'Los Angeles', 'Santa Barbara', 'Sacramento', 'Fresno','San Diego', 'San Luis Obispo']

def weather_plot(col, cities=cities):
    """
    plot weather data on a map 
    col: 'Temprature', 'Humidity', 'Wind Speed'
    """
    df = weather_data(cities)
    df['x'], df['y'] = lnglat_to_meters(df['lon'], df['lat'])
    table = hv.Table(df[['name', col]]).opts(width=800)
    points = df.hvplot.scatter('x','y', c=col, cmap='bkr', hover_cols=['name'])
    map_tiles  = EsriImagery().opts(alpha=0.5, width=900, height=480, bgcolor='white')
    return  pn.Column(points * map_tiles, table)
    
 
 class refresh_weather_dashboard(param.Parameterized):
    action = param.Action(lambda x: x.param.trigger('action'), label='Refresh')
    select_column = param.ObjectSelector(default='Temprature', objects=['Temprature', 'Humidity', 'Wind Speed'])
    
    @param.depends('action', 'select_column')
    def get_plot(self):
        return weather_plot(self.select_column)

    
weather_dashboard = refresh_weather_dashboard()

pn_weather = pn.Column(
       pn.panel(weather_dashboard.param, show_labels=True, show_name=False, margin=0),
       weather_dashboard.get_plot, width=400
)

pane = pn.Tabs(
    ('Real Time', pn_realtime),
    ('Refresh Weather Dashboard', pn_weather)
    ).servable()

pane


# Para ejecutarlo
panel serve realtime_dashboard.py


#  Links a tutoriales
https://towardsdatascience.com/building-a-real-time-dashboard-using-python-plotly-library-and-web-service-145f50d204f0

https://sophiamyang.github.io/DS/visualization/real-time/realtimedash.html

https://dev.to/shittu_olumide_/how-to-use-build-a-live-dashboard-with-python-5ed0

https://coderzcolumn.com/tutorials/data-science/how-to-create-dashboard-using-python-matplotlib-panel

https://hackernoon.com/build-a-real-time-stock-price-dashboard-with-python-questdb-and-plotly

https://medium.com/initial-state/how-to-stream-push-data-to-a-real-time-dashboard-api-in-python-7a910e41c001

# Dashboard en 7 minutos
https://www.natasshaselvaraj.com/interactive-dashboard-python/

https://blog.streamlit.io/how-to-build-a-real-time-live-dashboard-with-streamlit/

https://www.statworx.com/en/content-hub/blog/how-to-build-a-dashboard-in-python-plotly-dash-step-by-step-tutorial/

