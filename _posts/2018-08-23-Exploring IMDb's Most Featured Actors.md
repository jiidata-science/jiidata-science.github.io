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

So here it is...my first article (hopefully not my last). I decided to kick-off my series with a look at [IMDb's Top 250 rated movies](https://www.imdb.com/chart/top). More specifically, in this article, we scrape IMDb's top rated movies, along with their corresponding cast & crew listings, and explore who the "real movie superstars" are - essentially, *which actors feature in the most/multiple top rated movies?* 

I began this project with the naive assumption that these "superstars" would all be well-known household names; the likes of *Katharine Hepburn, Robert De Niro* and *Jack Nicholson* - similar to IMDb's list of [100 greatest actors & actresses of all time](https://www.imdb.com/list/ls053085147/) - **BUT this analysis resulted in some suprising finds!**

For this project we used Python, and a selection of libraries, for data collection, exploration and visualisation. This article includes a selection of Python code-blocks and visualisations that help us to explore and draw insights from the data. The full codeset is available on [Github](https://github.com/jiidata-science/Imdb_Top_Actors) as a Jupyter notebook.

### Motivation

So why did I chose to explore this random and niche topic? Well, primarily I wanted to demonstrate use of web scraping technology to collect data that wasn't otherwise readily available in the desired format (yawn!). The second reason was merely because I enjoy watching movies - *hardly a unique interest* - so, I was inately interested in the data itself.

### What we'll be looking at

  * <a href="#part1">Part 1</a> (of 5): Importing Required Python Libraries
  * <a href="#part2">Part 2</a> (of 5): Data collection - Scraping IMDb's Top 250 Rated Movies, [using BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
  * <a href="#part3">Part 3</a> (of 5): Data collection - Scraping Movie Genre & Full Cast + Crew, [using BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
  * <a href="#part4">Part 4</a> (of 5): Data Exploration - Visualising Movie Ratings, with [Pandas](https://pandas.pydata.org/) and [Plotly](https://plot.ly/python/)
  * <a href="#part5">Part 5</a> (of 5): Data Exploration - *Who Really Are The Best Actors?*, with [Pandas](https://pandas.pydata.org/) and [Plotly](https://plot.ly/python/)


*Additional note: whilst IMDb does NOT readily offer APIs for accessing movie information (which seemed a little suprising to me) they do offer a number of [static datasets](https://datasets.imdbws.com/). I chose **not** use these datasets and scraped what I needed directly from the IMDb.com.*

<a name="part1"></a>
## Part 1 of 5: Importing Required Python Libraries

This step is pretty trivial so little explanation is needed, in addition to the commented code-block below. It is perhaps useful to note, however, that in order to use Plotly, within a jupyter notebook, I had to use the ```plotly.offline``` configuration and specify ```init_notebook_mode(connected=True)``` .

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

<a name="part2"></a>
## Part 2 (of 5): Data collection - Scraping IMDb's Top 250 Rated Movies
*Data captured on 27th July 2018*

We begun collecting data using *urlib3* and *beautifulSoup* libraries to scrape the Top 250 movies from IMDb's [Top 250 movie charts](https://www.imdb.com/chart/top). We captured each movie title, along with it's official IMDb ranking and rating. All the data we required could be found within the html table with **class = 'chart full-width'**, on the IMDb web page (highlighted in the printscreen below).

![alt text](https://raw.githubusercontent.com/jiidata-science/Imdb_Top_Actors/master/Images/TopRatedMovies.png "Top Rated Movies Table")

The code block, below, was created to scrape the tabulated data and store it as lists in the ```table_data``` list object.

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
  
Printing the first few values stored in *table_data* so you can see the output data.

    table_data[:5] # print the top 5 results

    #[Output:]
    #Top250Rank | MovieName | Published | Rating
    #---------- | --------- | --------- | ------
    #1|'The Shawshank Redemption'|1994|9.2
    #2|'The Godfather'|1972|9.2
    #3|'The Godfather: Part II'|1974|9.0
    #4|'The Dark Knight'|2008|9.0
    #5|'12 Angry Men'|1957|8.9

<a name="part3"></a>
## Part 3 (of 5): Data collection - Scraping Movie Genre & Full Cast + Crew

Having captured all of the top rated movie names, and some additional information, I then focused on capturing the **genre** and **full cast and crew**, for each movie. This data allowed us to explore actors that featured across multiple top rated movies.

> Before explaining this process any further, it's important to note that scraping web page data, using html tags, quite obviously relies on the website maintaining a consistent canonical html structure. It is likely that certain sites (most likely popular sites with non-static content) will look to optimise user-journeys and page layout over time. Even in the two week period I was looking at this topic, I had to modify my scraping code in accordance with updates to html tags on the requested web pages. Just be mindful of this, particularly if you're planning to routinely schedule your scraping process.

On IMDb.com, each listed movie has its own landing page, covering a summary of movie information, and a separate page for viewing the corresponding full cast & crew. Providing that you use the IMDb 'film_id' (i.e. a bespoke ID that IMDb have created to uniquely store movie-level data) it's very simple to manipulate a couple of IMDb page URLs to retrieve the information we're after:

 - **https://www.imdb.com/title/{film_id}** *: used to retrieve film genre. Replace {film_id} with integer movie ID value* (example: **https://www.imdb.com/title/tt0111161** for Shawshank Redemption).

 - **https://www.imdb.com/title/{film_id}/fullcredits** *: used to retrieve full cast & crew. Replace {film_id} with integer movie ID value* (example: **https://www.imdb.com/title/tt0111161/fullcredits** for Shawshank Redemption full cast & crew).

We actually already had the 'film_IDs' in the web page requested in <a href="#part2">Part 2</a>  - we just hadn't stored them yet. These ids could be found in the html table structure within the **class = 'wlb_ribbon'**. Once we'd captured all film_IDs, we iteratively constructed the movie-specific URLs and scraped the data we were after, illustrated in the code block, below.

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

I wrote a code statement, above, that printed to the log for each 25th iteration (just so we had an indication of progress).

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

Having completed our data collection, we had the following local datasets:

1. ```python table_data```: contains the following attributes for each movie ([movie_rank, movie_name, movie_year, movie_rating])
2. ```film_ids```: contains the following attributes for each movie ([filmID , filmID_castURL , filmID_homePageURL, genre_clean])
2. ```base_castAndCrew```: contains the full name for each full cast & crew member, for each movie (e.g. [['Robert DeNiro', 'Julia Roberts']])

Now we were able to start exploring!

<a name="part4"></a>
## Part 4 (of 5): Data Exploration - Visualising Movie Ratings

I began by exploring the movie ratings themselves; asking questions of the data such as **Is there a linear relationsip between movie ranking and movie rating?** and **Which movie genres typically have higher ratings?**.

The boxplot below illustrates the distribution of IMDb movie ratings. With a median movie rating of 8.2 and an upper fence of 8.8, the boxplot identifies seven 'outliers' that have anomalously high ratings. To no suprise, these top rated films include well-known favourites - including [1] The Shawshank Redemption, [2] The Godfather, [3] The Godfather: Part II, [4] The Dark Knight, and [5] 12 Angry Men.

<iframe width="900" height="500" frameborder="0" scrolling="no" src="//plot.ly/~jii-datascience/4.embed"></iframe>

Extending on the above point, movie ratings do increase linearly with movie rankings. The cumulative distribution, below, presents the distribution of movie ratings slightly differently. Form this chart we deduce that 95% of movies have a rating between 8.2 - 8.7 (range: 0.5), whilst the remaining 5% have much higher scores between 8.7 to 9.2 (range: 0.5)

<iframe width="900" height="600" frameborder="0" scrolling="no" src="//plot.ly/~jii-datascience/6.embed"></iframe>

<a name="part5"></a>
## Part 5 (of 5): Data Exploration - *Who Really Are The Best Actors?*

So this was the bit I was most interested in. Here we take a look at how many, top rated, movies each actor appeared in.

Firstly, I looked at the frequency of movie features per actor. Each actor was assigned to a frequency bucket (i.e. 1, 2, 3, 4+) depending on the number of top rated movies they'd featured in. 

The pie chart below illustrates this split, for the full **15,013 (distinct) cast & crew members** that featured across the 250 movies. From the chart we deduce that **90% of actors featured in just one movie**, with just **0.88% of actors featuring in 4 or more movies**. Interesting? Perhaps not, but I then started to put names to these numbers.

You can find the code for this visualisation on [Github](https://github.com/jiidata-science/Imdb_Top_Actors).

<iframe width="900" height="550" frameborder="0" scrolling="no" src="//plot.ly/~jii-datascience/8.embed"></iframe>

We then created an interactive plotly chart that allows the user to see the actors with the highest movie features, with the list of movies available on hover. I chose to only plot actors that had featured in atleast five movies as anymore and the plot would be unreadable (and I was too lazy to created a input range or value filter). The plot starts begins with the highest number of features on the far left of the chart, progressing in descending order.

<iframe width="950" height="580" frameborder="0" scrolling="no" src="//plot.ly/~jii-datascience/12.embed"></iframe>

So what did I deduce from this chart? Well, perhaps I'm not quite the film buff I first thought but:

- to my suprise, the first actor I had heard of was in position five - Robert De Niro - who has appeared in eight of the top 250 films (and is undoubtably a household name).

- in position one, **John Ratzenberger** has 'appeared' in more of the Top 250 movies than any other 'actor' - a whopping 12 movies. Interetingly, **10 of these were animation films**, so he didn't even 'appear' in them at all.

- **Bess Flowers**, in position two, featured in 10 movies all published before the 1970s.

- in third position, **Joseph Oliveira**, was a peculiar find. Whilst he'd featured in 9 of the Top 250 movies, he'd only played **supporting** or **'uncredited'** roles in each. To list a few examples, Joseph featured as a [walk on officer in Dark Knight (2008)](https://www.imdb.com/title/tt0468569/fullcredits?ref_=tt_cl_sm#cast); held an uncredited role as ['Marciano' in Goodfellas (1990)](https://www.imdb.com/title/tt0099685/fullcredits?ref_=tt_cl_sm#cast); and **again** an [uncredited Officer Court Room Attendant in Wolf of Wall Street (2013)](https://www.imdb.com/title/tt0993846/fullcredits). He's been in so many great modern films but there's no chance I'd know who he was if he passed me on the street.

- there are also plently of expected actors, including Harrison Ford (6th), Gino Corrado (7th), Morgan Freeman (10th) to name a few.

You can find the code for this visualisation on [Github](https://github.com/jiidata-science/Imdb_Top_Actors).

  
## Summary & Final Thoughts
  
