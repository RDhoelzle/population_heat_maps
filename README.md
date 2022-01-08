# Normalising Population Data for Geographical Heat Maps

It's a well-known challenge when visualisating geographical data that if the data isn't handled properly, then geographical heatmaps tend to be no more than population maps.

<img src="https://imgs.xkcd.com/comics/heatmap_2x.png" width="500">
<div style="text-align: center"> <a href="https://xkcd.com/1138/" target="_blank">XKCD Heat Maps</a> </div>

This is due to simple sampling bias, ie: places with higher population density are more likely to have people with a given characteristic. As data scientists, we tell stories about data using visualisations, and population maps are seldom interesting or informative.

This effect can be seen on a global scale when plotting immigration trends. The most populous countries (China, India, and the United States) tend to produce the greatest number immigrants, especially to developed countries. Let's explore this phenomenon in the context of immigration to Canada, then normalise the data by population to see how it changes the story.

## 1. Setup python environment

First, we'll setup the python environment. The immigration and population data are in excel files, which I'm reading to dataframes using openpyxl (`!pip3 install openpyxl`). We'll visualise our data with `Folium`.

```python
#import libraries
import wget
import os.path
from zipfile import ZipFile
import geojson
import numpy as np
import pandas as pd
import folium
```

## 2. Import and process the immigration data

The UN Population Division tracks and reports global migration patterns in their *International Migration to and from Selected Countries* series. In the 2015 release, used here, time series data is reported from 1980-2013 for 45 countries. The country-by-country breakdown can be obtained <a href="https://www.un.org/en/development/desa/population/migration/data/empirical2/migrationflows.asp#" target="_blank">here</a>.

After downloading the zip file, extract the *Canada.xlsx* file into your working directory with `ZipFile`. If you don't want to download the full zip file, I've posted the Canada file for download <a href="https://github.com/RDhoelzle/population_heat_maps/blob/main/Data/Canada.xlsx" target="_blank">here</a>.

The country-by-country immigration data is contained in the *Canada by Citizenship* sheet. We'll skip the first 20 and last 2 rows, which are copyright information and yearly and global totals, respectively. We'll also restrict our data to the first 43 columns with the `usecols="A:AQ"` parameter to avoid loading the series of blank columns after the 2013 data.

```python
#download and extract immigration data
if not os.path.isfile('Data/Canada.xlsx'):
    if not os.path.isfile('UN_MigFlow_All_CountryFiles.zip'):
        url = 'https://www.un.org/en/development/desa/population/migration/data/empirical2/data/UN_MigFlow_All_CountryFiles.zip'
        wget.download(url)
    with ZipFile('UN_MigFlow_All_CountryFiles.zip', 'r') as zipObj:
        zipObj.extract('Data/Canada.xlsx')
```

```python
#import emigration data
immi = pd.read_excel(
    'Data/Canada.xlsx',
    sheet_name='Canada by Citizenship',
    usecols="A:AQ",
    skiprows=range(20), #the first 20 rows contain copyright information
    skipfooter=2) #the final 2 rows are yearly and global totals

immi.head()
```

|       |       Type |   Coverage |         OdName | AREA | AreaName |  REG |         RegName | DEV |            DevName | 1980 | ... | 2004 | 2005 | 2006 | 2007 | 2008 | 2009 | 2010 | 2011 | 2012 | 2013 |
| -----:| ----------:| ----------:| --------------:| ----:| --------:| ----:| ---------------:| ---:| ------------------:| ----:| ---:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:|
| **0** | Immigrants | Foreigners |    Afghanistan |  935 |     Asia | 5501 |   Southern Asia | 902 | Developing regions |   16 | ... | 2978 | 3436 | 3009 | 2652 | 2111 | 1746 | 1758 | 2203 | 2635 | 2004 |
| **1** | Immigrants | Foreigners |        Albania |  908 |   Europe |  925 | Southern Europe | 901 |  Developed regions |    1 | ... | 1450 | 1223 |  856 |  702 |  560 |  716 |  561 |  539 |  620 |  603 |
| **2** | Immigrants | Foreigners |        Algeria |  903 |   Africa |  912 | Northern Africa | 902 | Developing regions |   80 | ... | 3616 | 3626 | 4807 | 3623 | 4005 | 5393 | 4752 | 4325 | 3774 | 4331 |
| **3** | Immigrants | Foreigners | American Samoa |  909 |  Oceania |  957 |       Polynesia | 902 | Developing regions |    0 | ... |    0 |    0 |    1 |    0 |    0 |    0 |    0 |    0 |    0 |    0 |
| **4** | Immigrants | Foreigners |        Andorra |  908 |   Europe |  925 | Southern Europe | 901 |  Developed regions |    0 | ... |    0 |    0 |    1 |    1 |    0 |    0 |    0 |    0 |    1 |    1 |

`5 rows x 43 columns`

For this project, we're really only concerned with the country of origin and the yearly total immigration. To make later data processing easier, we'll drop the rest of the metadata columns (`["Type","Coverage","AREA","AreaName","REG","RegName","DEV","DevName"]`).

```python
#drop irrelevant columns
immi.drop(["Type","Coverage","AREA","AreaName","REG","RegName","DEV","DevName"], axis=1, inplace=True)

#rename 'OdName' to 'Country'
immi.rename(columns={'OdName': 'Country'}, inplace=True)

#set year columns to numeric and replace blank data with 0s
for col in list(immi.columns.values):
    if col != 'Country':
        immi[col] = pd.to_numeric(immi[col], errors='coerce', downcast='integer').fillna(0).astype(int)

immi.head()
```

|       |        Country | 1980 | 1981 | 1982 | 1983 | 1984 | 1985 | 1986 | 1987 | 1988 | ... | 2004 | 2005 | 2006 | 2007 | 2008 | 2009 | 2010 | 2011 | 2012 | 2013 |
| -----:| --------------:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ---:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:|
| **0** |    Afghanistan |   16 |   39 |   39 |   47 |   71 |  340 |  496 |  741 |  828 | ... | 2978 | 3436 | 3009 | 2652 | 2111 | 1746 | 1758 | 2203 | 2635 | 2004 |
| **1** |        Albania |    1 |    0 |    0 |    0 |    0 |    0 |    1 |    2 |    2 | ... | 1450 | 1223 |  856 |  702 |  560 |  716 |  561 |  539 |  620 |  603 |
| **2** |        Algeria |   80 |   67 |   71 |   69 |   63 |   44 |   69 |  132 |  242 | ... | 3616 | 3626 | 4807 | 3623 | 4005 | 5393 | 4752 | 4325 | 3774 | 4331 |
| **3** | American Samoa |    0 |    1 |    0 |    0 |    0 |    0 |    0 |    1 |    0 | ... |    0 |    0 |    1 |    0 |    0 |    0 |    0 |    0 |    0 |    0 |
| **4** |        Andorra |    0 |    0 |    0 |    0 |    0 |    0 |    2 |    0 |    0 | ... |    0 |    0 |    1 |    1 |    0 |    0 |    0 |    0 |    1 |    1 |

`5 rows x 35 columns`

Great! Now let's calculate the total immigration for each country into a final `"Total"` column, which will be the final value that we plot to the heat map.

```python
#calculate the total immigration from each country
immi["Total"] = immi.iloc[:, immi.columns != 'Country'].sum(axis=1)

immi.head()
```

|       |        Country | 1980 | 1981 | 1982 | 1983 | 1984 | 1985 | 1986 | 1987 | 1988 | ... | 2005 | 2006 | 2007 | 2008 | 2009 | 2010 | 2011 | 2012 | 2013 | Total |
| -----:| --------------:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ---:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| -----:|
| **0** |    Afghanistan |   16 |   39 |   39 |   47 |   71 |  340 |  496 |  741 |  828 | ... | 3436 | 3009 | 2652 | 2111 | 1746 | 1758 | 2203 | 2635 | 2004 | 58639 |
| **1** |        Albania |    1 |    0 |    0 |    0 |    0 |    0 |    1 |    2 |    2 | ... | 1223 |  856 |  702 |  560 |  716 |  561 |  539 |  620 |  603 | 15699 |
| **2** |        Algeria |   80 |   67 |   71 |   69 |   63 |   44 |   69 |  132 |  242 | ... | 3626 | 4807 | 3623 | 4005 | 5393 | 4752 | 4325 | 3774 | 4331 | 69439 |
| **3** | American Samoa |    0 |    1 |    0 |    0 |    0 |    0 |    0 |    1 |    0 | ... |    0 |    1 |    0 |    0 |    0 |    0 |    0 |    0 |    0 |     6 |
| **4** |        Andorra |    0 |    0 |    0 |    0 |    0 |    0 |    2 |    0 |    0 | ... |    0 |    1 |    1 |    0 |    0 |    0 |    0 |    1 |    1 |    15 |

`5 rows x 36 columns`

## 3. Plot a heat map of the raw immigration data

Ok, we're done with the initial data processing and ready to plot a heat map. Geographic heat maps are also called `Choropleth` maps. These require a geojson file that defines the geographic boundaries of each country. There are a number of open source options for this file. For this project, we'll use the *countries.geojson* file provided in *GitHub's Open Data* repository. Several of the country names in this file don't match those in the immigration data file, so we'll used an edited version which I've posted <a href="https://github.com/RDhoelzle/public_projects/blob/master/Normalised%20Population%20Heat%20Maps/Data/countries.geojson" target="_blank">here</a>.

Some countries, such as Côte d'Ivoire, contain special or accented characters in their names. We'll convert the text in the geojson file to utf-8 encoding to ensure that these countries exactly match the immigration data.

```python
#download countries geojson file
if not os.path.isfile('Data/countries.geojson'):
    url = 'https://github.com/RDhoelzle/population_heat_maps/blob/main/Data/countries.geojson'
    wget.download(url)

#convert accented characters to utf-8
with open('Data/countries.geojson', encoding='utf-8') as file:
    map_shapes = geojson.load(file)
```

To generate the heat map, we'll start by creating a `map` object centered at 25&deg;N, 5&deg;E (to make sure New Zealand is plotted). We'll also set the zoom to 2, which displays one full standard Mercator projection.

We'll expand the number of bins in the colourscale to 7 by defining 8 evenly spaced bin edges from 0 to 1+ the maximum total immigration. We'll then call the `choropleth` method with the country names and total immigration by country from our cleaned data, as well as the geojson file.

```python
#create map object
tot_immi_map = folium.Map(location=[25, 5], zoom_start=2)

#create an array of length 8 with linear spacing from 0 to the maximum total immigration + 1
scale = np.linspace(0, immi['Total'].max()+1, 8, dtype=int)
scale = scale.tolist()

#add the color mapping features
folium.Choropleth(
    data=immi,
    columns=["Country", "Total"], #country names and total immigration by country
    geo_data=map_shapes, #geojson file
    key_on='feature.properties.ADMIN', #path to country names in geojson file
    threshold_scale=scale,
    bins=7,
    fill_color='RdPu',
    fill_opacity=0.7,
    line_opacity=0.2,
    nan_fill_color='white',
    nan_fill_opacity=0.7,
    legend_name='Total Immigration to Canadia, 1980-2013'
).add_to(tot_immi_map)

# display map
tot_immi_map
```
<img src="https://github.com/RDhoelzle/population_heat_maps/blob/main/Images/tot_immi_map.jpg?raw=true" width="700">

4 of the 6 most immigrating countries also rank in the top 5 most populous (China, India, United States, and Pakistan). The Philippines and United Kingdom are outliers, though both are still highly populous countries.

We can compare this map to the 1980-2013 average population using the *Total Population - Both Sexes* data from the UN Population Division available <a href="https://population.un.org/wpp/Download/Standard/Population/" target="_blank">here</a>. We'll reduce this data set to only the years matching the immigration data, remove world region names, and adjust "United Kingdom" and "Czechia" to match their immigration data names of "United Kingdom of Great Britain and Northern Ireland" and "Czech Republic", respectively.

```python
#download world population
if not os.path.isfile('Data/WPP2019_POP_F01_1_TOTAL_POPULATION_BOTH_SEXES.xlsx'):
    url = 'https://population.un.org/wpp/Download/Files/1_Indicators%20(Standard)/EXCEL_FILES/1_Population/WPP2019_POP_F01_1_TOTAL_POPULATION_BOTH_SEXES.xlsx'
    wget.download(url)

#import population data
population = pd.read_excel(
    'Data/WPP2019_POP_F01_1_TOTAL_POPULATION_BOTH_SEXES.xlsx',
    sheet_name='ESTIMATES',
    usecols="A:BZ",
    skiprows=range(16), #the first 16 rows contain copyright information
)

population.head()
```

|       | Index |   Variant | Region, subregion, country or area \* | Notes | Country code |              Type | Parent code |        1950 |        1951 |        1952 | ... |        2011 |        2012 |        2013 |        2014 |        2015 |        2016 |       2017 |        2018 |        2019 |        2020 |
| -----:| -----:| ---------:| -------------------------------------:| -----:| ------------:| -----------------:| -----------:| -----------:| -----------:| -----------:| ---:| -----------:| -----------:| -----------:| -----------:| -----------:| -----------:| ----------:| -----------:| -----------:| -----------:|
| **0** |     1 | Estimates |                                 WORLD |   NaN |          900 |             World |           0 | 2536431.018 | 2584034.227 |  2630861.69 | ... | 7041194.168 | 7125827.957 | 7210582.041 | 7295290.759 | 7379796.967 | 7464021.934 |  7547858.9 | 7631091.113 | 7713468.205 | 7794798.729 |
| **1** |     2 | Estimates |                 UN development groups |     a |         1803 |   Label/Separator |         900 |         ... |         ... |         ... | ... |         ... |         ... |         ... |         ... |         ... |         ... |        ... |         ... |         ... |         ... |
| **2** |     3 | Estimates |                More developed regions |     b |          901 | Development Group |        1803 |  814818.913 |  824003.512 |  833720.173 | ... | 1239557.448 | 1244114.531 |  1248453.53 | 1252615.112 | 1256622.188 | 1260478.667 | 1264146.38 | 1267558.904 |  1270630.32 | 1273304.261 |
| **3** |     4 | Estimates |                Less developed regions |     c |          902 | Development Group |        1803 | 1721612.105 | 1760030.715 | 1797141.517 | ... |  5801636.72 | 5881713.426 | 5962128.511 | 6042675.647 | 6123174.779 | 6203543.267 | 6283712.52 | 6363532.209 | 6442837.885 | 6521494.468 |
| **4** |     5 | Estimates |             Least developed countries |     d |          941 | Development Group |         902 |  195427.785 |  199180.385 |  203015.198 | ... |  856471.437 |  876867.234 |  897793.439 |  919222.955 |  941131.317 |  963519.718 | 986385.402 | 1009691.252 | 1033388.868 | 1057438.163 |

`5 rows x 78 columns`

```python
#remove regions and subregions
population = population[population['Country code'] < 900]

#drop irrelevant columns
population.drop(["Index","Variant","Notes","Country code","Type","Parent code"], axis=1, inplace=True)
population.drop([str(x) for x in list(range(1950,1980))], axis=1, inplace=True)
population.drop([str(x) for x in list(range(2014,2021))], axis=1, inplace=True)

#rename 'Region, subregion, country or area *' to 'Country'
population.rename(columns={'Region, subregion, country or area *': 'Country'}, inplace=True)

#fix UK and Czechia to match immigration and geojson
population.loc[(population.Country == "United Kingdom"), "Country"] = "United Kingdom of Great Britain and Northern Ireland"
population.loc[(population.Country == "Czechia"), "Country"] = "Czech Republic"

population.head()
```

|        |  Country |      1980 |      1981 |      1982 |      1983 |      1984 |      1985 |      1986 |      1987 |      1988 | ... |      2004 |     2005 |      2006 |      2007 |      2008 |      2009 |      2010 |      2011 |      2012 |      2013 |
| ------:| --------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---:| ---------:| --------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:|
| **26** |  Burundi |  4157.296 |   4266.52 |  4379.727 |  4497.544 |  4621.096 |  4750.832 |  4886.745 |  5027.143 |  5168.703 | ... |  7131.688 | 7364.857 |   7607.85 |  7862.226 |  8126.104 |  8397.661 |  8675.606 |  8958.406 |  9245.992 |  9540.302 |
| **27** |  Comoros |   307.831 |   317.617 |   326.944 |   336.088 |   345.455 |   355.337 |   365.765 |   376.647 |   387.964 | ... |    597.23 |  611.625 |   626.427 |   641.624 |   657.227 |   673.251 |   689.696 |   706.578 |   723.865 |   741.511 |
| **28** | Djibouti |    358.96 |   374.934 |   385.268 |     393.8 |   406.018 |   425.608 |   454.359 |   490.337 |   528.993 | ... |   771.599 |  783.248 |   794.554 |   805.456 |   816.361 |    827.82 |   840.194 |   853.671 |   868.136 |   883.296 |
| **29** |  Eritrea |  1733.423 |  1784.557 |  1836.825 |  1890.556 |  1946.299 |  2003.942 |  2064.803 |  2127.421 |  2185.607 | ... |  2719.809 | 2826.653 |  2918.209 |   2996.54 |  3062.782 |   3119.92 |  3170.437 |  3213.969 |  3250.104 |  3281.453 |
| **30** | Ethiopia | 35141.703 | 35984.531 | 36995.246 | 38142.679 | 39374.346 | 40652.146 | 41965.696 | 43329.238 | 44757.205 | ... | 74239.508 | 76346.31 | 78489.205 | 80674.343 | 82916.236 | 85233.923 | 87639.962 | 90139.928 | 92726.982 | 95385.793 |

`5 rows x 35 columns`

Finally, we'll calculate the average population (in millions for easier scaling) for each country and plot the heat map.

```python
#calculate the average population from each country
#the raw data is in 1000s, so we'll divide by 1000 to get final data in millions
population["Average"] = population.iloc[:, population.columns != 'Country'].mean(axis=1)/1000

population.head()
```

|        |  Country |      1980 |      1981 |      1982 |      1983 |      1984 |      1985 |      1986 |      1987 |      1988 | ... |     2005 |      2006 |      2007 |      2008 |      2009 |      2010 |      2011 |      2012 |      2013 |   Average |
| ------:| --------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---:| --------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:| ---------:|
| **26** |  Burundi |  4157.296 |   4266.52 |  4379.727 |  4497.544 |  4621.096 |  4750.832 |  4886.745 |  5027.143 |  5168.703 | ... | 7364.857 |   7607.85 |  7862.226 |  8126.104 |  8397.661 |  8675.606 |  8958.406 |  9245.992 |  9540.302 |  6.338221 |
| **27** |  Comoros |   307.831 |   317.617 |   326.944 |   336.088 |   345.455 |   355.337 |   365.765 |   376.647 |   387.964 | ... |  611.625 |   626.427 |   641.624 |   657.227 |   673.251 |   689.696 |   706.578 |   723.865 |   741.511 |  0.503910 |
| **28** | Djibouti |    358.96 |   374.934 |   385.268 |     393.8 |   406.018 |   425.608 |   454.359 |   490.337 |   528.993 | ... |  783.248 |   794.554 |   805.456 |   816.361 |    827.82 |   840.194 |   853.671 |   868.136 |   883.296 |  0.645651 |
| **29** |  Eritrea |  1733.423 |  1784.557 |  1836.825 |  1890.556 |  1946.299 |  2003.942 |  2064.803 |  2127.421 |  2185.607 | ... | 2826.653 |  2918.209 |   2996.54 |  3062.782 |   3119.92 |  3170.437 |  3213.969 |  3250.104 |  3281.453 |  2.423324 |
| **30** | Ethiopia | 35141.703 | 35984.531 | 36995.246 | 38142.679 | 39374.346 | 40652.146 | 41965.696 | 43329.238 | 44757.205 | ... | 76346.31 | 78489.205 | 80674.343 | 82916.236 | 85233.923 | 87639.962 | 90139.928 | 92726.982 | 95385.793 | 61.293582 |

`5 rows x 35 columns`

```python
#create map object
pop_map = folium.Map(location=[25, 5], zoom_start=2)

#create an array of length 8 with linear spacing from 0 to the maximum average population + 1
scale = np.linspace(0, population['Average'].max()+1, 8, dtype=int)
scale = scale.tolist()

#add the color mapping features
folium.Choropleth(
    data=population,
    columns=["Country", "Average"], #country names and total immigration by country
    geo_data=map_shapes, #geojson file
    key_on='feature.properties.ADMIN', #path to country names in geojson file
    threshold_scale=scale,
    bins=7,
    fill_color='RdPu',
    fill_opacity=0.7,
    line_opacity=0.2,
    nan_fill_color='white',
    nan_fill_opacity=0.7,
    legend_name='World Mean Population (millions), 1980-2013'
).add_to(pop_map)

# display map
pop_map
```
<img src="https://github.com/RDhoelzle/population_heat_maps/blob/main/Images/pop_map.jpg?raw=true" width="700">

## 4. Normalise the immigration data by population and plot

As you can see, the immigration and population heat maps are identical apart from a few outliers. The fact that the United Kingdom produces a similar number of Canadian immigrants to China and India is interesting, though not necessarily surprising when we consider that Canada is part of the British Commonwealth.

Let's see how the story changes by normalising the immigration data. Here, we'll perform a type of *feature normalisation* by dividing the total immigration from each country by that country's average population (both from 1980 to 2013), generating an *immigration rate*, or immigration per population.

*Immigration_rate* = *Immigration_total*/*Population_average*

Demographic rates are often expressed per ten thousand, hundred thousand, or million people, depending on the data scale. We've already scaled our average populatin data to millions, which happens to result in a convenient scale for our final immigration rate, so we'll maintain that scale for this last step.

We'll need to do a little more data cleaning prepair the two datasets for normalisation. First, sampling bias can also occur when normalising data from small countries (<10M), so we'll remove those countries form our population data. Next, we'll make sure both datasets are in the same order by sorting by country name. Finally, we'll then subset the two datasets by the other's country list and verify that the lists are identical.

```python
#remove low population countries
population = population[population['Average'] >= 10]

#sort countries by alphabetical order
immi.sort_values(by=['Country'], inplace=True)
population.sort_values(by=['Country'], inplace=True)

#subset dataframes to same list of countries
immi = immi[immi['Country'].isin(population.Country)]
population = population[population['Country'].isin(immi.Country)]

print("Shape of Immigration data:", immi.shape)
print("Shape of Population data:", population.shape)

#check the country lists
if list(immi.Country) == list(population.Country):
    print("The Country lists are identical")
else:
    print("The Country lists don't match")
```
```
Shape of Immigration data: (74, 36)
Shape of Population data: (74, 36)
The Country lists are identical
```

Now that our two datasets are identical, we'll calculate the immigration rate and add it as a new column in the immigration dataset and plot as before.

```python
#scale total immigration per 100,000 average population
immi["Rate"] = np.divide(list(immi['Total']),list(population['Average']))

immi.head()
```

|       |     Country | 1980 | 1981 | 1982 | 1983 | 1984 | 1985 | 1986 | 1987 | 1988 | ... | 2006 | 2007 | 2008 | 2009 | 2010 | 2011 | 2012 | 2013 | Total |        Rate |
| -----:| -----------:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ---:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| -----:| -----------:|
| **0** | Afghanistan |   16 |   39 |   39 |   47 |   71 |  340 |  496 |  741 |  828 | ... | 3009 | 2652 | 2111 | 1746 | 1758 | 2203 | 2635 | 2004 | 58639 | 3004.084614 |
| **2** |     Algeria |   80 |   67 |   71 |   69 |   63 |   44 |   69 |  132 |  242 | ... | 4807 | 3623 | 4005 | 5393 | 4752 | 4325 | 3774 | 4331 | 69439 | 2400.943139 |
| **5** |      Angola |    1 |    3 |    6 |    6 |    4 |    3 |    5 |    5 |   11 | ... |  184 |  106 |   76 |   62 |   61 |   39 |   70 |   45 |  2113 |  136.260820 |
| **7** |   Argentina |  368 |  426 |  626 |  241 |  237 |  196 |  213 |  519 |  374 | ... |  847 |  620 |  540 |  467 |  459 |  278 |  263 |  282 | 19596 |  555.769560 |
| **9** |   Australia |  702 |  639 |  484 |  317 |  317 |  319 |  356 |  467 |  410 | ... |  875 | 1033 | 1018 | 1018 |  933 |  851 |  982 | 1121 | 23829 | 1291.258590 |

`5 rows x 37 columns`


```python
#create map object
scaled_immi_map = folium.Map(location=[25, 5], zoom_start=2)

#create an array of length 8 with linear spacing from 0 to the maximum total immigration + 1
scale = np.linspace(0, immi['Rate'].max()+1, 8)
scale = scale.tolist()

#add the color mapping features
folium.Choropleth(
    data=immi,
    columns=["Country", "Rate"], #country names and total immigration by country
    geo_data=map_shapes, #geojson file
    key_on='feature.properties.ADMIN', #path to country names in geojson file
    threshold_scale=scale,
    bins=7,
    fill_color='RdPu',
    fill_opacity=0.7,
    line_opacity=0.2,
    nan_fill_color='white',
    nan_fill_opacity=0.7,
    legend_name='Canadian Immigration Rates (per million), 1980-2013'
).add_to(scaled_immi_map)

# display map
scaled_immi_map
```
<img src="https://github.com/RDhoelzle/population_heat_maps/blob/main/Images/scaled_immi_map.jpg?raw=true" width="700">

## 5. Conclusions

Our previous map highlighted the Commonwealth relationship between Canada and the United Kingdom, as well as the fact that populous countries tend to produce a large number of immigrants. This map tells a completely different story. The United Kingdom still stands out, as does the Philippines. However, we can now see that a number of Middle Eastern, European, North African, and Latin American countries generate a disproportionaly large number of immigrants. The reasons why fall under three main categories, and are far more interesting than raw population size or historical ties.

#### Refugees

The Middle East, North Africa, and Latin America have been wracked with instability for decades. Prolonged wars in Iran, Iraq, Afghanistan, Algeria, and Colombia have resulted in expansive immigrant populations around the globe, and Canada has long welcomed a <a href="https://www.macleans.ca/news/canada/refugee-resettlement-canada/" target="_blank">disproportionately high number of refugees</a>. People likewise fled the Chilean Military Dictatorship in the 1970s and 80s, as well as the Syrian Civil War starting in 2010.

#### Poor Economic Opportunities

A primary reason that Canada is able to welcome so many refugees is its strong and stable economy. This also makes Canada an attractive destination for immigrants looking for economic opportunity. In Portugal, for instance, the expensive Colonial Wars of the 1960s and 1970s, followed by a series of revolutionary coups in the mid 1970s, resulted in a stagnant economy through the 1980s. Polish and Romanians also emmigrated for better economic opportunities following the fall of the Soviet Union in 1991.

#### Overpopulation

Overpopulation drove high immigration rates in the most of the remaining countries. A specific subset of poor economic opportunities, operpopulation occurs when a country's birth rate outpaces its ability to create both jobs and homes. This was the primary cause for immigration from the Philippines, Sri Lanka, Pakistan, and South Korea.

Viewing Canadian immigraion data this way, we can see that Canada not only accepts a diverse range of immigrants from around the globe, but that its strong economy and commitment to liberal policies makes it an ideal country for immigrants to build a better life.
