---
title: Plotly - Advanced
date: 2023-08-05 02:40:12
tags:
- Python
- Data Visualization
- Plotly
---

# For Example: Line Plots
📘 Download [World Earthquake Data From 1906-2022](https://www.kaggle.com/datasets/garrickhague/world-earthquake-data-from-1906-2022)

---
## Line Plots with `plotly.express`
*<font color=royalblue>The plotly.express module (usually imported as px) contains functions that can create entire figures at once. 
Plotly Express is a built-in part of the plotly library, and is the recommended starting point for creating most common figures.</font>*

```python
import pandas as pd
import plotly.express as px

df = pd.read_csv('data.csv')

# Pre-processing
df['Year'] = pd.to_datetime(df['time']).dt.year
df["Country"] = df["place"].str.split(pat=',', expand=False).str.get(-1)
df["Max_mag"] = df.groupby(['Year'])['mag'].transform(max)

df = df.query("2012 <= Year <= 2022")
fig = px.line(df, x="Year", y="Max_mag", title='Max magnitude from 2000 to 2022')
fig.show()
```

- Explanation
  - Pandas `groupby` splits all the records from your data set into different categories or groups and offers you flexibility to analyze the data by these groups.

    ```python
    # Group by 'Year', and get the maximum of 'mag' column
    df.groupby(['Year'])['mag'].max()

    # Group by 'Year', get the maximum of 'mag' column, and fit the length of dataframe  
    df.groupby(['Year'])['mag'].transform(max)

    # Query '2022 >= Year >= 2021' and get the value of 'Max_mag' column
    df.query("2022 >= Year >= 2021")["Max_mag"]
    ```

---
## Line Plots with Column Encoding Color
```python
import pandas as pd
import plotly.express as px

df = pd.read_csv('data.csv')

# Pre-processing
df['Year'] = pd.to_datetime(df['time']).dt.year
df["Country"] = df["place"].str.split(pat=',', expand=False).str.get(-1)
df["Max_mag"] = df.groupby(['Year', 'magType'])['mag'].transform(max)

df = df.query("2012 <= Year <= 2022")
fig = px.line(df, x="Year", y="Max_mag", 
  title='Max magnitude from 2012 to 2022', color='magType')
fig.show()
```

- Explanation
  - Pandas `groupby` splits all the records from your data set into different categories or groups and offers you flexibility to analyze the data by these groups.

    ```python
    # Group by 'Year' and 'magType', and get the maximum of 'mag' column
    df.groupby(['Year', 'magType'])['mag'].max()

    # Group by 'Year' and 'magType', get the maximum of 'mag' column, and fit the length of dataframe  
    df.groupby(['Year', 'magType'])['mag'].transform(max)

    # Query '2022 >= Year >= 2021' and get the value of 'Max_mag' column
    df.query("2022 >= Year >= 2021")["Max_mag"]
    ```

---
## Basic Settings

### Line charts with Markers
- Set Title
  - `title = 'Max magnitude and Min depth from 2012 to 2022'`
- Add one more field on text label
  - `text = "Min_depth"`

    ```python
    import pandas as pd
    import plotly.express as px

    df = pd.read_csv('data.csv')

    # Pre-processing
    df['Year'] = pd.to_datetime(df['time']).dt.year
    df["Country"] = df["place"].str.split(pat=',', expand=False).str.get(-1)
    df["Max_mag"] = df.groupby(['Year', 'magType'])['mag'].transform(max)
    df["Min_depth"] = df.groupby(['Year', 'magType'])['depth'].transform(min)

    query_df = df.query("2022 >= Year >= 2012 & magType in ['mw', 'ml', 'ms', 'mb']")
    fig = px.line(query_df, x="Year", y="Max_mag", 
      title='Max magnitude and Min depth from 2012 to 2022', 
      color='magType', text="Min_depth")
    fig.show()
    ```

- Mark data points
  - Method 1: `markers = True`
  - Method 2: `symbol = "magType"`

    ```python
    import pandas as pd
    import plotly.express as px

    df = pd.read_csv('data.csv')

    # Pre-processing
    df['Year'] = pd.to_datetime(df['time']).dt.year
    df["Country"] = df["place"].str.split(pat=',', expand=False).str.get(-1)
    df["Max_mag"] = df.groupby(['Year', 'magType'])['mag'].transform(max)

    query_df = df.query("2022 >= Year >= 2012")

    # Method 1: markers
    fig = px.line(query_df, x="Year", y="Max_mag", 
      title='Max magnitude from 2012 to 2022', 
      color='magType', markers=True)
    fig.show()

    # Method 2: symbol
    fig = px.line(query_df, x="Year", y="Max_mag", 
      title='Max magnitude from 2012 to 2022', 
      color='magType', symbol="magType")
    fig.show()
    ```

### Save a Plot
```basic
# Install Package
$ pip install -U kaleido
```

- Save a Figure in xxx Format: PNG, JPEG, SVG, PDF
    ```python
    # PNG
    fig.write_image("fig1.png")

    # JPEG
    fig.write_image("fig1.jpeg")

    # SVG
    fig.write_image("fig1.svg")

    # PDF
    fig.write_image("fig1.pdf")
    ```

- Change Image dimension and Scale
    ```python
    fig.write_image('fig2_scale2.jpeg', width=600, height=350, scale=2)
    ```

- Interactive HTML export in Plotly Python
    ```python
    fig.write_html("fig1.html", auto_open=True)
    ```

---
## Advanced: Interactive Custom Controls

### Basic Range Slider and Range Selectors
```python
import plotly.graph_objects as go
import pandas as pd

# Load data
df = pd.read_csv('data.csv')

# Pre-processing
df['Year'] = pd.to_datetime(df['time']).dt.year
df["Max_mag"] = df.groupby(['Year', 'magType'])['mag'].transform(max)

# Query
query_df = df.query("magType == 'mww'")

# Create figure
fig = go.Figure()

fig.add_trace(
    go.Scatter(x=list(query_df.time), y=list(query_df.Max_mag)))

# Set title
fig.update_layout(
    title_text="Time series with range slider and selectors"
)

# Add range slider
fig.update_layout(
    xaxis=dict(
        rangeselector=dict(
            buttons=list([
                dict(count=1, label="1m", step="month", stepmode="backward"),
                dict(count=6, label="6m", step="month", stepmode="backward"),
                dict(count=1, label="YTD", step="year", stepmode="todate"),
                dict(count=1, label="1y", step="year", stepmode="backward"),
                dict(step="all")
            ])
        ),
        rangeslider=dict( visible=True ),
        type="date"
    )
)

fig.show()
```

---
# References
- [USGS - Magnitude Types](https://www.usgs.gov/programs/earthquake-hazards/magnitude-types)
- [矩量級 Moment Magnitude Scale](https://academic-accelerator.com/encyclopedia/zh/moment-magnitude-scale)
- [Magnitude Type: Mw vs. ML vs. MS vs. mb](https://www.facebook.com/momlovestaiwan/photos/a.1076309132504673/1076350115833908/?type=3&locale=zh_TW)
- [Plotly Open Source Graphing Library for Python](https://plotly.com/python/)