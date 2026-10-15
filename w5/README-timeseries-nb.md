---
jupytext:
  encoding: '# -*- coding: utf-8 -*-'
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.19.5
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# time series, the basics

```{admonition} to download
:class: warning

to work on this assignment locally, {download}`start by downloading the zip<./ARTEFACTS-timeseries.zip>`
```

In this assignment we study **cycling in Paris**, using the hourly counts of one of the
bike counters that the city has installed in its streets.

This is a *first contact* with time series in pandas; we will practice

* **parsing** strings into timestamps, and dealing with time zones
* the **`.dt` accessor**, to extract the hour, the day of the week, etc.
* **slicing** a dataframe on its time index
* **`resample()`**, to aggregate by day, week, month
* and finally **joining** with another time series, daily weather data

the companion assignment *time series: events* deals with series that are **not** on a regular grid

here is the kind of pictures we will end up with

<img src="media/hourly-profile.svg" width=500px>

```{code-cell} ipython3
import pandas as pd
import matplotlib.pyplot as plt

print(f"using pandas {pd.__version__}")

# time series look better in a wide figure
plt.rcParams["figure.figsize"] = (10, 4)
```

```{code-cell} ipython3
# optional
import itables
itables.init_notebook_mode()
```

## 0. the data

+++

there are two files in the `data/` folder

| file | one row per | columns |
|-|-|-|
| `bike-counter.csv` | hour | `timestamp`, `count` (number of bikes that went through the counter) |
| `weather.csv` | day | `jour`, `temperature` (°C, daily mean), `rain_mm` (daily total) |

the counter is on the *voie Georges Pompidou*, along the Seine; it counts the bikes going in one direction only  
we start with the first one

+++

## 1. parsing the timestamps

+++

### loading

+++

1. load `data/bike-counter.csv` into a dataframe named `df`
1. look at the first rows and at the types of the columns - what is the type of the `timestamp` column ?
1. the name `count` is unfortunate, because `df.count` is a method of the dataframe;  
   rename that column into `n_bikes`

```{code-cell} ipython3
:tags: [level_basic]

df = pd.read_csv("data/bike-counter.csv")
print(df.dtypes)
df.rename(columns = {"count" : "n_bikes"}, inplace = True)
df.head()
```

### converting

+++

we want to turn the `timestamp` column into actual timestamps

1. use `pd.to_datetime()` for that; what do you get ? read the message carefully
1. look at the strings in the data: apart from the date and the hour, what else is there ?  
   how many different values are there for that part ? can you guess why ?

```{code-cell} ipython3
:tags: [level_basic]

df.timestamp = pd.to_datetime(df.timestamp, utc = True)
#df.timestamp.value_counts()
df.dtypes
```

:::{admonition} hint
:class: tip dropdown

the last 6 characters of a string can be obtained with `df.timestamp.str[-6:]`  
and then you can use `value_counts()`
:::

+++

### converting, again

+++

pandas has a hard time with a column that mixes several time zone offsets  
the way out is to ask for everything to be expressed in UTC, using the `utc=True` option

1. redo the conversion with that option, and store the result back into the `timestamp` column;  
   what is the dtype now ?
1. we are interested in the **local time** in Paris, since this is when people go to work  
   convert that column to the timezone `"Europe/Paris"`  
   you will need to find the right method in the `.dt` accessor (or do a search on `tz_convert`)

```{code-cell} ipython3
:tags: [level_basic]

df.timestamp = df.timestamp.dt.tz_convert(tz="Europe/Paris")
df.timestamp.head()
```

## 2. the `.dt` accessor

+++

now that `timestamp` has a proper type, the `.dt` accessor gives us access to all the details  
add these columns to `df`

| column | content |
|-|-|
| `hour` | the hour of the day, 0 to 23 |
| `weekday` | the day of the week, as a string like `"Monday"` |
| `is_weekend` | a boolean, `True` on Saturday and Sunday |
| `month` | the month, as a number 1 to 12 |

```{code-cell} ipython3
:tags: [level_basic]

df['Hour'] = df.timestamp.dt.hour
df['Weekday'] = df.timestamp.dt.day_name()
df['Is_wee'] =  (df['Weekday'] == "Sunday") | (df['Weekday'] == "Saturday")
df["Month"] = df.timestamp.dt.month
df
```

:::{admonition} hint
:class: tip dropdown

* `.dt.hour`, `.dt.month` are attributes
* `.dt.day_name()` is a method, and there is also an attribute `.dt.dayofweek` that returns a number with 0 for Monday
:::

+++

### the hourly profile

+++

1. what is the average number of bikes **per hour of the day** ? use a `groupby` and plot the result with `.plot()`  
   what are the busiest hours ? does that match your expectations ?
1. now do the same but separately for week days and for week ends, so that both curves show on the same plot  
   we expect something like the picture in the introduction

:::{admonition} hint
:class: tip dropdown

for the second question, group by 2 criteria, and think about `unstack()` to get 2 columns  
this is the same trick as what we have seen with `pivot_table`
:::

```{code-cell} ipython3
:tags: [level_basic]

av_by_hour = df.groupby(by = "Hour")["n_bikes"].mean()
av_by_hour.plot()
```

## 3. using the time as the index

+++

it is very common to use the timestamps as the index of a time series; this makes slicing very convenient

1. make `timestamp` the index of `df`
1. make sure the index is sorted in chronological order (check that with `index.is_monotonic_increasing`)
1. what are the first and the last timestamps ? so, how long is the period covered ?

```{code-cell} ipython3
:tags: [level_basic]

df.set_index("timestamp", inplace = True)
print(df.index.is_monotonic_increasing)
print(df.head(1))
print(df.tail(1)) # 12/09/25 --> 11/10/26
```

### missing data

+++

this is a counter that sits in the street, so there can be some missing data  
we expect one row per hour

1. how many rows would you expect between the first and the last timestamp ? how many do we have ?
1. how many rows contain `NaN` in the `n_bikes` column ?
1. which days are entirely missing ?

:::{admonition} hint
:class: tip dropdown

* `pd.date_range()` can create the index with all the hours; use the option `freq="h"`
* or use `df.asfreq("h")`, that creates the missing rows with `NaN` in it
:::

```{code-cell} ipython3
:tags: [level_basic]

print(df.shape) # 9465 = 24*(365 + 29) + 9
print(df["n_bikes"].isna().sum())
compar = pd.date_range("2025-09-12 14:00:00+02:00", "2026-10-11 23:00:00+02:00", freq = 'h')
compar.difference(df.index).to_series().dt.date.value_counts()
```

## 4. slicing

+++

now that we have a time index, we can slice with `.loc[]` using strings that look like dates  
let us practice, using `df.loc[...]` to answer these questions

1. what are the total number of bikes counted in December 2025 ?
1. how many rows do we have between January 1st and January 7th 2026 (both included) ?  
   is the end of the slice included or not ?
1. how many bikes in total in the morning rush, i.e. between 7:00 and 10:00 (all days included) ?
1. how many rows do we have on 2026-03-29 ? and on 2025-10-26 ? why is that ?

:::{admonition} hint
:class: tip dropdown

for question 3, you can build a mask from the hour of the day, which is available
as an attribute of the index (`df.index.hour`) or from the column we have created earlier  
and if you like, there is also a method `between_time()` that does it directly
:::

```{code-cell} ipython3
:tags: [level_basic]

print(df.loc["2025-12-01 00:00:00+02:00" : "2025-12-31 23:00:00+02:00"]["n_bikes"].sum())
print(df.loc["2026-01-01 00:00:00+02:00" : "2026-01-07 23:00:00+02:00"]["n_bikes"].sum())
print(df[(7<= df['Hour']) & (10>=df['Hour'])]["n_bikes"].sum())
```

### is the practice growing ? (optional)

+++

our data covers the period from September 12th to October 11th **twice**: in 2025 and in 2026  
compare the total number of bikes in each of these 2 periods

```{code-cell} ipython3
:tags: [level_basic]

df.loc["2025-09-12 00:00:00+02:00" : "2025-10-11 23:00:00+02:00"]["n_bikes"].sum()<=df.loc["2026-09-12 00:00:00+02:00" : "2026-10-11 23:00:00+02:00"]["n_bikes"].sum()
```

## 5. resampling

+++

`resample()` is like a `groupby`, where the groups are time intervals; it is the tool of choice to change the granularity of a time series  
so, to move from one point per hour to one point per day

1. compute `daily`, the total number of bikes per day (a `Series`), and plot it
1. what is the busiest day ? and the least busy one ?  
   hint: `idxmax()` and `idxmin()`

```{code-cell} ipython3
:tags: [level_basic]

daily = df.resample("D")["n_bikes"].sum()
print("busiest : ", daily.idxmax())
print("least busy : ", daily.idxmin())
daily.plot()
```

### beware of the missing data

+++

some days have **no bike at all**, which is suspicious - recall that some days were missing entirely  
on the plot above they show as a few sharp drops down to 0, in the late summer and in the fall

1. which are these days ? what happened ? what does `sum()` return on a group with no data at all ?
Pour les 3 jours sans données, sum() retourne 0
1. fix `daily` so that these days show up as `NaN` rather than 0  
   hint: look at the options of `sum()`
1. plot again, but only the last 2 months, and with `marker="."` to see the individual points;  
   what does `plot()` do with a `NaN` ?

```{code-cell} ipython3
:tags: [level_basic]

daily = df.resample("D")["n_bikes"].sum(min_count = 1)
daily.plot(marker = '.') # Les NaN ne sont pas représentés
daily
```

### weekly and monthly

+++

1. compute and plot the number of bikes per week, and per month  
   in the resulting series, what is the first date for a month ? and for a week ?
1. the first and last months are not complete; in addition, the months have different lengths  
   so a monthly total is not the best way to compare them; compute the **average per day** instead  
   hint: you can use `daily`
1. make a dataframe with 3 columns `total`, `mean` and `max` giving, for each month, the corresponding aggregates of the daily numbers  
   hint: `agg()`

```{code-cell} ipython3
:tags: [level_basic]

weekly = df.resample("W")["n_bikes"].sum() # first date : 2025-09-14
monthly = df.resample("ME")["n_bikes"].sum() # first date : 2025-09-30
print(daily.mean())
daily.groupby(by = [daily.index.year, daily.index.month]).agg(["sum", "mean", "max"])
```

:::{admonition} hint
:class: tip dropdown

the aliases for the frequency are `"W"` for weeks, and `"ME"` for the end of the months  
(`"MS"` for the start of the months); in older versions of pandas it was `"M"`
:::

+++

## 6. adding the weather

+++

does the weather make a difference ? let us find out with the second file

1. load `data/weather.csv` into a dataframe `weather`, and make sure the column `jour` becomes a proper timestamp  
   careful with the format: `12/09/2025` is meant to be September 12th; what would happen if pandas chose to read it as December 9th ?  
   use the `format=` option of `pd.to_datetime` to say what you mean
1. make that column the index of `weather`
1. combine `daily` and `weather` into one dataframe `both`, with the 3 columns `n_bikes`, `temperature` and `rain_mm`  
   at that point you may run into an issue with the indexes - that is because one has a time zone, and the other one has not  
   we recommend to remove the time zone from `daily`, you can use `tz_localize(None)` on its index

```{code-cell} ipython3
:tags: [level_basic]

weather = pd.read_csv("data/weather.csv")
jour = pd.to_datetime(weather.jour, format = "%d/%m/%Y")
weather.set_index(jour, inplace = True)
both = weather.join(daily.tz_localize(None))
```

### a look at the result

+++

1. are there less bikes on rainy days ?  
   let us say a day is rainy if there is more than 1mm of rain  
   compare the average number of bikes on rainy days and on other days
1. draw a scatter plot of the number of bikes against the temperature, using `df.plot.scatter()` or similar
1. what is the correlation between the 3 columns ?  
   hint: `corr()`

```{code-cell} ipython3
:tags: [level_basic]

print("rainy", both[both["rain_mm"]>=1.0]["n_bikes"].mean())
print("not rainy", both[both["rain_mm"]<1.0]["n_bikes"].mean()) # less bikes on rainy days
both.plot("temperature", "n_bikes", kind = "scatter")
both[["temperature", "rain_mm", "n_bikes"]].corr()
```

## appendix: where the data comes from

+++

* the bike counts come from the open data portal of the city of Paris, dataset *comptage vélo - données compteurs*  
  <https://opendata.paris.fr>
* the weather comes from the *Open-Meteo* archive API, for Paris  
  <https://open-meteo.com>
* the script `fetch-data.py`, that comes with this assignment, was used to acquire this data
  - it picks one counter, and exports the hours in local time **with the UTC offset**, as the original does
  - and it writes the days of the weather in the French format on purpose
* the data is frozen, no need to refresh it
