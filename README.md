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

|     |       Type |   Coverage |         OdName | AREA | AreaName |  REG |         RegName | DEV |            DevName | 1980 | ... | 2004 | 2005 | 2006 | 2007 | 2008 | 2009 | 2010 | 2011 | 2012 | 2013 |
| ---:| ----------:| ----------:| --------------:| ----:| --------:| ----:| ---------------:| ---:| ------------------:| ----:| ---:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:| ----:|
|   0 | Immigrants | Foreigners |    Afghanistan |  935 |     Asia | 5501 |   Southern Asia | 902 | Developing regions |   16 | ... | 2978 | 3436 | 3009 | 2652 | 2111 | 1746 | 1758 | 2203 | 2635 | 2004 |
|   1 | Immigrants | Foreigners |        Albania |  908 |   Europe |  925 | Southern Europe | 901 |  Developed regions |    1 | ... | 1450 | 1223 |  856 |  702 |  560 |  716 |  561 |  539 |  620 |  603 |
|   2 | Immigrants | Foreigners |        Algeria |  903 |   Africa |  912 | Northern Africa | 902 | Developing regions |   80 | ... | 3616 | 3626 | 4807 | 3623 | 4005 | 5393 | 4752 | 4325 | 3774 | 4331 |
|   3 | Immigrants | Foreigners | American Samoa |  909 |  Oceania |  957 |       Polynesia | 902 | Developing regions |    0 | ... |    0 |    0 |    1 |    0 |    0 |    0 |    0 |    0 |    0 |    0 |
|   4 | Immigrants | Foreigners |        Andorra |  908 |   Europe |  925 | Southern Europe | 901 |  Developed regions |    0 | ... |    0 |    0 |    1 |    1 |    0 |    0 |    0 |    0 |    1 |    1 |

5 rows x 43 columns

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

Great! Now let's calculate the total immigration for each country into a final `"Total"` column, which will be the final value that we plot to the heat map.

```python
#calculate the total immigration from each country
immi["Total"] = immi.iloc[:, immi.columns != 'Country'].sum(axis=1)

immi.head()
```

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

Finally, we'll calculate the average population (in millions for easier scaling) for each country and plot the heat map.

```python
#calculate the average population from each country
#the raw data is in 1000s, so we'll divide by 1000 to get final data in millions
population["Average"] = population.iloc[:, population.columns != 'Country'].mean(axis=1)/1000

population.head()
```

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

## 4. Normalise the immigration data by population and plot

As you can see, the immigration and population heat maps are identical apart from a few outliers. The fact that the United Kingdom produces a similar number of Canadian immigrants to China and India is interesting, though not necessarily surprising when we consider that Canada is part of the British Commonwealth.

Let's see how the story changes by normalising the immigration data. Here, we'll perform a type of *feature normalisation* by dividing the total immigration from each country by that country's average population (both from 1980 to 2013), generating an *immigration rate*, or immigration per population.
<br>
<br>
<div style="text-align: center"> ${Immigration}_{rate} = \frac{{Immigration}_{total}} {{Population}_{average}} $ </div>

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

Now that our two datasets are identical, we'll calculate the immigration rate and add it as a new column in the immigration dataset and plot as before.

```python
#scale total immigration per 100,000 average population
immi["Rate"] = np.divide(list(immi['Total']),list(population['Average']))

immi.head()
```

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

## 5. Conclusions

Our previous map highlighted the Commonwealth relationship between Canada and the United Kingdom, as well as the fact that populous countries tend to produce a large number of immigrants. This map tells a completely different story. The United Kingdom still stands out, as does the Philippines. However, we can now see that a number of Middle Eastern, European, North African, and Latin American countries generate a disproportionaly large number of immigrants. The reasons why fall under three main categories, and are far more interesting than raw population size or historical ties.

* Refugees
The Middle East, North Africa, and Latin America have been wracked with instability for decades. Prolonged wars in Iran, Iraq, Afghanistan, Algeria, and Colombia have resulted in expansive immigrant populations around the globe, and Canada has long welcomed a <a href="https://www.macleans.ca/news/canada/refugee-resettlement-canada/" target="_blank">disproportionately high number of refugees</a>. People likewise fled the Chilean Military Dictatorship in the 1970s and 80s, as well as the Syrian Civil War starting in 2010.

* Poor Economic Opportunities
A primary reason that Canada is able to welcome so many refugees is its strong and stable economy. This also makes Canada an attractive destination for immigrants looking for economic opportunity. In Portugal, for instance, the expensive Colonial Wars of the 1960s and 1970s, followed by a series of revolutionary coups in the mid 1970s, resulted in a stagnant economy through the 1980s. Polish and Romanians also emmigrated for better economic opportunities following the fall of the Soviet Union in 1991.

* Overpopulation
Overpopulation drove high immigration rates in the most of the remaining countries. A specific subset of poor economic opportunities, operpopulation occurs when a country's birth rate outpaces its ability to create both jobs and homes. This was the primary cause for immigration from the Philippines, Sri Lanka, Pakistan, and South Korea.

Viewing Canadian immigraion data this way, we can see that Canada not only accepts a diverse range of immigrants from around the globe, but that its strong economy and commitment to liberal policies makes it an ideal country for immigrants to build a better life.
