---

layout: single
date: 2018-08-23T11:48:41-04:00
header:
  overlay_color: "#000"
  overlay_filter: "0.6"
  overlay_image: /assets/images/site/DataBlog_titleImage.png
excerpt: "Scraping movie and cast information from IMDb.com and visualising most featured actors/actresses, with BeautifulSoup, Pandas and Plotly"
author_profile: true
author: Jez Irving
classes: wide

---

# Intro

So here it is...my first data science article (hopefully not my last). I decided to kick-off my series with at IMDb's top rated films to identify actors/actresses that feature across multiple top films - the 'real movie stars'. To be brutally honest about why I've dived into this relatively random topic; primarily I wanted to use web scraping technology (i.e. Python with beautifulSoup in this case) to collect data that isn't otherwise readily available in the desired format. The secondary reason was merely because I enjoy watching movies - hardly a unique interest.

Whilst IMDb's does readily offer APIs for accessing movie information (seems a little suprising to me) they do offer a number of [static datasets] (https://datasets.imdbws.com/). I chose not use these datasets and scraped required data directly from the IMDb.com.

In this blog:

  * We start with scraping IMDb film, actors/actress data (using BeautifulSoup)
  * We process and clean the captured data (using Pandas)
  * Then (more interestingly) we start to pull explore the data by:
    * Looking at the distribution of Top 250 film ratings
    * Understanding which film genres are more likely to have higher ratings, and
    * Identifying which actors/actresses appear in the most top rated films

Ultimately, this blog culminates in identifying "Which actors/actresses feature in the most Top 250 films?". I began this project under the naive assumption that these actors/actresses would be popular household names; the likes of Katharine Hepburn, Robert De Niro and Jack Nicholson. IMDb themselves provide a ranking of ['100 greatest actors & actresses'] (https://www.imdb.com/list/ls053085147/) BUT This analysis actually provides some suprising outcomes...enjoy!


``` python

# Import required Python libraries and setup workspace

import certifi
import urllib3
http = urllib3.PoolManager(cert_reqs='CERT_REQUIRED', ca_certs=certifi.where())
from bs4 import BeautifulSoup
import pandas as pd
import numpy as np
import csv

from scipy import polyfit, polyval
from scipy.interpolate import CubicSpline

import plotly.graph_objs as go
from plotly import __version__
from plotly.offline import download_plotlyjs, init_notebook_mode, plot, iplot
init_notebook_mode(connected=True)
%matplotlib inline

```

# DATA COLLECTION Part One
## Scraping Top 250 rated movies
* Data captured on 27th July 2018 *

We start by using urlib3 and beautifulSoup libraries to pull the Top 250 movies from IMDb's [Top 250 web page] (https://www.imdb.com/chart/top). We capture each movie title, along with it's official IMDb ranking and rating. In the code cell below, we create a list of lists, stored in the table_data variable.

``` python
table_data = [] # empty list. We'll be adding results to this

# IMDb page url for all top 250 rated films
url_imdbTop250 = 'https://www.imdb.com/chart/top'

# run web page request
page_imdbTop250 = http.request('GET', url_imdbTop250)

# allow for page exploration using BeautifulSoup (i.e. soup-ify returned webpage)
soup_Top250 = BeautifulSoup(page_imdbTop250.data, "lxml")

# create tabulated data with top 250 film information
table = soup_Top250.find('table', attrs={'class':'chart full-width'})# select the <table ...> that contains the ranked movies
table_body = table.find('tbody') # further subset the page. select only the table body
rows = table_body.find_all('tr') # find all rows within top 250 rated movies table

# for each row in table extract the: rank, movie name, year published and Imdb rating
for row in rows:
    cols = row.find_all('td')
    cols = [ele.text.strip() for ele in cols]
    movie_name_string = cols[1]
    movie_rank = movie_name_string[:movie_name_string.index('.')] 
    movie_year = int(movie_name_string[-5:-1])
    movie_name = movie_name_string[movie_name_string.index('.')+1:movie_name_string.index('(')].strip()
    movie_rating = cols[2]
    table_data.append([movie_rank, movie_name, movie_year, movie_rating])

table_data[:5] # print the top 5 results

```
  
  # [Output:]
  # Top250Rank | MovieName | Published | Rating
  # ---------- | --------- | --------- | ------
  # 1|'The Shawshank Redemption'|1994|9.2
  # 2|'The Godfather'|1972|9.2
  # 3|'The Godfather: Part II'|1974|9.0
  # 4|'The Dark Knight'|2008|9.0
  # 5|'12 Angry Men'|1957|8.9


# DATA COLLECTION Part Two
## Scrape movie Genre & Cast/Crew

Each movie has it's own title landing page, covering a summary of information and a separate page for viewing the full cast/crew list per feature. Providing that you use the Imdb movie Id (i.e. an ID specific to Imdb) it's very simple to manipulate standard URLs and pull the information need:

 - https://www.imdb.com/title/{film_id} * : used to retrieve film genre *
 - https://www.imdb.com/title/{film_id}/fullcredits *: used to retrieve full cast / crew*

The film IDs are first retrieved from the initial page scrape, as they are provided in the html in the **'wlb_ribbon'** class. With these 250 x film IDs we proceed to scrape the information we require. I have clearly commented the code, below, to ensure that it can be intuitively understood.




