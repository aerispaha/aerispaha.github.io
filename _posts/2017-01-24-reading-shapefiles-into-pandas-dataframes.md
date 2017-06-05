---
layout: post
title: Reading Shapefiles into Pandas Dataframes
---

I've just about had it up to here with and ArcMap and arcpy. Today I begin my quest to free myself from ever needing to rely on ESRI for spatial analysis and mapping.

[Geopandas](http://geopandas.org/) seems great, but I have had a lot of trouble getting it installed and have therefore been hesitant to rely on it in any package I create. Instead, I've used the following snippet to read a shapefile into a Pandas dataframe for quick analysis. You will need the [pyshp](https://pypi.python.org/pypi/pyshp) package and [Pandas](http://pandas.pydata.org/). If you don't have these, install them via pip and you're ready to go:

```python
import shapefile #the pyshp module
import pandas as pd

#read file, parse out the records and shapes
shapefile_path = r'path/to/shapefile/'
sf = shapefile.Reader(shapefile_path)

#grab the shapefile's field names (omit the first psuedo field)
fields = [x[0] for x in sf.fields][1:]
records = sf.records()
shps = [s.points for s in sf.shapes()]

#write the records into a dataframe
shapefile_dataframe = pd.DataFrame(columns=fields, data=records)

#add the coordinate data to a column called "coords"
shapefile_dataframe = shapefile_dataframe.assign(coords=shps)
```

Now `shapefile_dataframe` has all of the input shapefile's records and geometry data.
