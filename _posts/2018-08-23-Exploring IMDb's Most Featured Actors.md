---

layout: single
date: 2018-08-23T11:48:41-04:00
header:
  overlay_color: "#000"
  overlay_filter: "0.6"
  overlay_image: /assets/images/site/DataBlog_titleImage.png
excerpt: "Scraping, exploring & visualising most featured actors in IMDb's Top 250 movies, with BeautifulSoup, Pandas & Plotly"
author_profile: true
author: Jez Irving
classes: wide

---

## Introduction & Purpose

So here it is...my first data science article (hopefully not my last). I decided to kick-off my series with a look at IMDb's film and cast listings, to identify which actors feature in the most top rated movies; to find out for myself who the 'real' movie stars are? This article includes as selection of Python code-blocks aswell as visualisations that help us to explore and understand the data.

So why have I chosen to explore this relatively random topic? Primarily, I wanted to demonstrate the use of web scraping technology to collect data that isn't otherwise readily available in the desired format. The second reason was merely because I enjoy watching movies...hardly a unique interest.

**In this blog:**

  * We start by scraping the Top 250 rated movies and all cast/crew, from IMDb.com (**[using BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)**)
  * We process and clean the scraped data (**[using Pandas](https://pandas.pydata.org/)**)
  * Then we start to **explore and visualise the data** (**[with Plotly](https://plot.ly/python/)**) by:
    * Looking at the distribution of Top 250 film ratings
    * Understanding which film genres are more likely to have higher ratings, and
    * Identifying which actors appear in the most top rated films

Ultimately, this blog culminates in identifying *which actors feature in the most Top 250 films?*. I began this project under the naive assumption that such actors would be well-known household names; the likes of *Katharine Hepburn, Robert De Niro* and *Jack Nicholson* - similar to IMDb's list of [100 greatest actors & actresses of all time](https://www.imdb.com/list/ls053085147/) - **BUT this analysis provides some suprising outcomes...enjoy!**

*Additional note: whilst IMDb does readily offer APIs for accessing movie information (which seemed a little suprising to me) they do offer a number of [static datasets](https://datasets.imdbws.com/). I chose **not** use these datasets and scraped what I needed directly from the IMDb.com.*

## Part 1 (of 5): Required Python Libraries
``` python

# libraries for requesting and scraping web pages
import certifi
import urllib3
http = urllib3.PoolManager(cert_reqs='CERT_REQUIRED', ca_certs=certifi.where())
from bs4 import BeautifulSoup

# libraries for structuring data
import pandas as pd
import numpy as np
import csv

# libraries for fitting models/relationships (exploratory analysis)
from scipy import polyfit, polyval
from scipy.interpolate import CubicSpline

# libraries (and configuration) for visualisation with Plotly
import plotly.graph_objs as go
from plotly import __version__
from plotly.offline import download_plotlyjs, init_notebook_mode, plot, iplot
init_notebook_mode(connected=True)
%matplotlib inline

```


## Part 2 (of 5): Data collection, scraping IMDb's Top 250 rated movies
*Data captured on 27th July 2018*

We start by using *urlib3* and *beautifulSoup* libraries to scrape the Top 250 movies from IMDb's [Top 250 web page](https://www.imdb.com/chart/top).
We capture each movie title, along with it's official IMDb ranking and rating. All required data is stored within the table **'chart full-width'** class attribute, on the IMDb web page, highlighted in the printscreen below.

![alt text](https://raw.githubusercontent.com/jiidata-science/Imdb_Top_Actors/master/Images/TopRatedMovies.png "Top Rated Movies Table")

The code block, below, scrapes the tabulated data and stores it as lists (in the table_data list object).

``` python
table_data = [] # empty list object. We'll be storing scraped data in this

# IMDb page url for Top 250 rated movies
url_imdbTop250 = 'https://www.imdb.com/chart/top'

# Executre web page request
page_imdbTop250 = http.request('GET', url_imdbTop250)

# Allow for page exploration using BeautifulSoup (i.e.'soupify' returned webpage)
soup_Top250 = BeautifulSoup(page_imdbTop250.data, "lxml")

# Subset returned webpage - select Top Rated Movies 'table' only
table = soup_Top250.find('table', attrs={'class':'chart full-width'})
table_body = table.find('tbody') # further subset the page. select only the table body
rows = table_body.find_all('tr') # find all rows within top 250 rated movies table

# For each row in table extract the: rank, movie name, year published and Imdb rating
for row in rows:
    cols = row.find_all('td')
    cols = [ele.text.strip() for ele in cols]
    movie_name_string = cols[1]
    movie_rank = movie_name_string[:movie_name_string.index('.')] 
    movie_year = int(movie_name_string[-5:-1])
    movie_name = movie_name_string[movie_name_string.index('.')+1:movie_name_string.index('(')].strip()
    movie_rating = cols[2]

    # for each row, append selected attributes to the table_data object
    table_data.append([movie_rank, movie_name, movie_year, movie_rating]) 
```
  
Print the first few values stored in *table_data*

    table_data[:5] # print the top 5 results

    #[Output:]
    #Top250Rank | MovieName | Published | Rating
    #---------- | --------- | --------- | ------
    #1|'The Shawshank Redemption'|1994|9.2
    #2|'The Godfather'|1972|9.2
    #3|'The Godfather: Part II'|1974|9.0
    #4|'The Dark Knight'|2008|9.0
    #5|'12 Angry Men'|1957|8.9


## Part 3 (of 5): Data collection, scraping movie genre & cast + crew

Having captured all Top Movies, and their corresponding Movie IDs in step 2, we now focus on capturing the movie **genre** and **cast and crew** for each movie. This data will allow us to explore actors that feature across multiple top rated movies.

On IMDb.com, each listed movie has it's own title landing page, covering a summary of movie information, and a separate page for viewing the corresponding full cast & crew. Providing that you use the Imdb movie Id (i.e. an ID specific to Imdb) it's very simple to manipulate IMDb's page URLs to retrieve the information we're after:

 - https://www.imdb.com/title/{film_id} *: used to retrieve film genre. Replace {film_id} with integer movie ID value*
  ⋅⋅⋅Example: https://www.imdb.com/title/tt0111161 for Shawshank Redemption

 - https://www.imdb.com/title/{film_id}/fullcredits *: used to retrieve full cast & crew. Replace {film_id} with integer movie ID value*
  ⋅⋅⋅Example: https://www.imdb.com/title/tt0111161/fullcredits for Shawshank Redemption full cast & crew

The **film IDs are first retrieved from the *soupified* page from step 2**, as they are provided in the html in the **'wlb_ribbon'** class. With these 250 x film IDs we proceed to scrape the genre and cast data we require. The code block, below, is clearly commented.

``` python
# Empty list objects. We'll be storing scraped data in this
film_ids = []
base_castAndCrew = []

# Soupified webpage from Step 2 (above). Retrieve all movie IDs
links_class  = soup_Top250.findAll("div", {"class":"wlb_ribbon"}) # table/html tag containing each IMDb movie ID

# For each movie, using the film_ID:
for idx, link in enumerate(links_class):    
    
    counter = idx + 1
    if counter % 25 == 0: # print to log every n iterations
        print("[INFO] Scraping film no. {0} of {1}".format(counter,len(links_class)))
        
    # Part 1 ----------------------------------------
    # FOR EACH MOVIE, USE MOVIE ID TO CONSTRUCT MOVIE PAGE URL AND RETURN FILM GENRE
    
    filmID = link.attrs['data-tconst']
    filmID_castURL = "https://www.imdb.com/title/{0}/fullcredits".format(filmID) # used to retrieve full cast
    filmID_homePageURL = "https://www.imdb.com/title/{0}".format(filmID) # used to retrieve genre
    
    # request movie home page to return genre
    html_filmHomePage = http.request('GET', filmID_homePageURL)
    soup_filmHomePage = BeautifulSoup(html_filmHomePage.data, "lxml")
    soup_getGenre = soup_filmHomePage.find('div', {'itemprop':'genre'})
    genre_clean = str(soup_getGenre.text).replace('\n' , '').replace('Genres: ', '')
    
    # append derived data to output list
    film_ids.append([filmID , filmID_castURL , filmID_homePageURL, genre_clean])
    
    # Part 2 ----------------------------------------
    # FOR EACH MOVIE, PULL FULL CAST FROM MOVIE CAST/CREW PAGE URL
    
    # query the website and return the html to the variable ‘page’
    html_castAndCrew = http.request('GET', filmID_castURL)
    soup_castAndCrew = BeautifulSoup(html_castAndCrew.data, "lxml")
    table_castAndCrew = soup_castAndCrew.find('table', {'class':'cast_list'})
    
    # select cast list
    list_castAndCrew = []
    for cast in table_castAndCrew.findAll('span', {'class':"itemprop"}):
        list_castAndCrew.append(cast.text)
    base_castAndCrew.append(list_castAndCrew)  
    
    del filmID, filmID_castURL, filmID_homePageURL, genre_clean, html_filmHomePage, soup_filmHomePage, soup_getGenre, \
    list_castAndCrew, html_castAndCrew, soup_castAndCrew, table_castAndCrew
``` 

We print to the log for each 25th iteration (just so we have an indication of progress).

    #[INFO] Scraping film no. 25 of 250
    #[INFO] Scraping film no. 50 of 250
    #[INFO] Scraping film no. 75 of 250
    #[INFO] Scraping film no. 100 of 250
    #[INFO] Scraping film no. 125 of 250
    #[INFO] Scraping film no. 150 of 250
    #[INFO] Scraping film no. 175 of 250
    #[INFO] Scraping film no. 200 of 250
    #[INFO] Scraping film no. 225 of 250
    #[INFO] Scraping film no. 250 of 250

At this point I all required data had been captured. We're now ready to start exploring!

We have two data sets:

1. **table_data** : contains the following attributes for each movie ([movie_rank, movie_name, movie_year, movie_rating])
2. **film_ids**: contains the following attributes for each movie ([filmID , filmID_castURL , filmID_homePageURL, genre_clean])
2. **base_castAndCrew**: contains the full name for each full cast & crew member, for each movie (e.g. [['Robert DeNiro', 'Julia Roberts']])


## Part 4 (of 5): Data Exploration, visualising rating distributions

I began by exploring the movie ratings themselves; asking questions of the data such as *Is there a linear relationsip between movie ranking and movie rating?* and *Which movie genres typically have higher ratings?*.

The boxplot below

<iframe width="900" height="500" frameborder="0" scrolling="no" src="//plot.ly/~jii-datascience/4.embed"></iframe>

Looking at the two ratings charts, we deduce that across the Top 250 movies:
  - the median movie rating is 8.2
  - The box plot identifies a number of outliers (note: Plotly atypically considers outliers as data points that are 1.5 x Interquartile range from the first (Q1) and third (Q3) quartiles. These seven outliers represent relatively (very) highly rated movies compares with others. With no suprise, these top rated films include some of my (and everyone elses) favouries
[{'1':'The Shawshank Redemption', '2': 'The Godfather','3': 'The Godfather: Part II','4': 'The Dark Knight', '5': '12 Angry Men'}]
-   extending on the above point, ratings do not have a purely (inverse) linear relationship with movie ranking. 95% of movies have a rating between 8.2 - 8.7 (range: 0.5), whilst the remaining 5% have much higher scores between 8.7 to 9.2 (range: 0.5).
  - The boxplot illustrates movie rating distribution characteristics, of the Top 250 rated films, split by film genre
  - I've ordered the x-axis from highest to lowest median rating value, by genre.
  - Perhaps, suprisingly, Music and Horror movies have the highest median rankings; but unsuprisgly each of these genres have just five contributing movies
  - X% of top movies are either dramas, x or y
  
## Part 5 (of 5): Data Exploration, *Who really are the best actors?*

  - So this was the bit I was most interested in. Here we take a look at which actors and actresses appear (in the cast & crew listings) across all 250 movies, how many movies they appeared in and what those films were?
  - in the code snippet below we create an interactive plotly chart that allows the user to select the top N actors/actresses with the most film features. The film start with the highest number of features appear on the far left of the chart and appears in descending order.
  - Perhaps I'm not quite the film buff I first thought but, to my suprise, the first actor I had heard of was in position five - Robert De Niro - who has appeared in eight of the top 250 films and is undoubtably a household name.
  - Interestingly, John Ratzenberger has 'appeared' in more of the Top 250 movies than any other actor - a whopping 12 x movies. However, 10 x of these were animation films - so he didn't even 'appear' in them at all. Bess Flowers, in position two, featured in 10 of the Top 250 movies but all before the 1970s.
  - In third position, Joseph Oliveira, was an quirky find - whilst he's featured in 9 of the Top 250 movies, he's only played 'supporting' or uncredited roles in each. For example: Dark Knight - a walk on officer, https://www.imdb.com/title/tt0468569/fullcredits?ref_=tt_cl_sm#cast uncredited role as 'Marciano' in Goodfellas https://www.imdb.com/title/tt0099685/fullcredits?ref_=tt_cl_sm#cast The Departed, again an uncredited Officer Court Room Attendant (uncredited) in Wolf of Wall Street https://www.imdb.com/title/tt0993846/fullcredits
  
## Summary & Final Thoughts
  