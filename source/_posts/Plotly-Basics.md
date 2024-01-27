---
title: Plotly - Basics
date: 2023-08-03 13:52:27
description: " "
tags:
- Python
- Data Visualization
- Plotly
---

> <h3><i><font color=saddlebrown>If you work with massive datasets, <br>you can use a library like Plotly, designed to handle massive datasets well.</font></i></h3>

---
# Before Doing Data Visualization

## Find Datasets
- [Kaggle Datasets](https://www.kaggle.com/datasets)
- [Google Dataset Search](https://datasetsearch.research.google.com/)
- [Taiwan Government's Open Data](https://data.gov.tw/)
- [The U.S. Government's Open Data](https://data.gov/)
- ...

## Prepared Environment
- Install packages
  - Plotly
  - Pandas
  - [JupyterLab](https://jupyter.org/install)

```basic
$ pip install plotly==5.15.0
$ pip install pandas
$ pip install jupyterlab
$ jupyter lab
```

## Load Dataset
📘 Download [World Earthquake Data From 1906-2022](https://www.kaggle.com/datasets/garrickhague/world-earthquake-data-from-1906-2022)

- `head()`
  - Shows the first n (the default is 5) rows
- `tail()`
  - The “opposite” method of head() is tail()
  - Shows the last n (5 by default) rows of the dataframe object
- `info()` 
  - Prints out a concise summary of the dataframe, including information about the index, data types, columns, non-null values, and memory usage
- `describe()`
  - Generates descriptive statistics, including those that summarize the central tendency, dispersion, and shape of the dataset’s distribution

```python
import pandas as pd

df = pd.read_csv('data.csv')

# Shows the first 5 rows
print(df.head())

# Shows the last 5 rows
print(df.tail())

# Concise summary of the dataframe
print(df.info())

# Descriptive statistics
print(df.describe())

# Save as `Year` field
df['Year'] = pd.to_datetime(df['time']).dt.year

# Save as `Country` field
df["Country"] = df["place"].str.split(pat=',', expand=False).str.get(-1)
```

*<font color=royalblue>After collection, most data requires some degree of cleaning or reformatting before it can be analyzed or used to create visualizations.</font>*

---
# Getting Started - [Plotly](https://plotly.com/python/)

## Line Charts
Line charts are used to convey changes over time.

```python
import plotly.express as px

df = px.data.gapminder().query("country=='Canada'")
fig = px.line(df, x="year", y="lifeExp", title='Life expectancy in Canada')
fig.show()
```

```python
import plotly.express as px

df = px.data.gapminder().query("continent=='Oceania'")
fig = px.line(df, x="year", y="lifeExp", color='country')
fig.show()
```

## Histogram
Use a histogram to visualize the frequency distribution of a single event over a certain time period.
*<font color=royalblue>A histogram is the graphical representation of quantitative data.</font>*

```python
import plotly.express as px

df = px.data.tips()
fig = px.histogram(df, x="total_bill")
fig.show()
```

## Bar Charts
*<font color=royalblue>The bar chart is the graphical representation of categorical data.</font>*

```python
import plotly.express as px

long_df = px.data.medals_long()
fig = px.bar(long_df, x="nation", y="count", color="medal", title="Long-Form Input")
fig.show()
```

## Scatter Plots
If you wanted to **highlight the relationship or correlations between two variables** (e.g. marketing spend and revenue, or hours of weekly exercise vs. cardiovascular fitness), you could use a scatter plot to see, at a glance, if one increases as the other decreases (or vice versa).

```python
import pandas as pd
import plotly.express as px

df = pd.read_csv('data.csv')
df['Year'] = pd.to_datetime(df['time']).dt.year
fig = px.scatter(df, x="Year", y="mag")
fig.show()
```

## Pie chart
```python
import plotly.express as px

df = px.data.tips()
fig = px.pie(df, values='tip', names='day')
fig.show()
```

## Maps
```python
import pandas as pd
import plotly.express as px

df = pd.read_csv('data.csv')

# Draw a map after doing `mag >= 7` query
fig = px.density_mapbox(df.query("mag >= 7"), lat='latitude', lon='longitude', z='mag', radius=10,
                        center=dict(lat=0, lon=180), zoom=0, mapbox_style="stamen-terrain")
fig.show()
```

```python
import pandas as pd
import plotly.express as px

df = pd.read_csv('data.csv')

# Do pre-processing on `place` field 
df["Country"] = df["place"].str.split(pat=',', expand=False).str.get(-1)

fig = px.scatter_mapbox(df.query("mag >= 7"), lat="latitude", lon="longitude", 
                        hover_name="Country", hover_data=["mag", "depth"],
                        color_discrete_sequence=["red"], zoom=3, height=300)
fig.update_layout(mapbox_style="open-street-map")
fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0})
fig.show()
```

---
# References
- [Plotly Python Open Source Graphing Library Fundamentals](https://plotly.com/python/plotly-fundamentals/)
- [Python Plotly Express Tutorial: Unlock Beautiful Visualizations](https://www.datacamp.com/tutorial/python-plotly-express-tutorial)
