---

layout: single
date: 2018-08-23T11:48:41-04:00
header:
  overlay_color: "#000"
  overlay_filter: "0.6"
  overlay_image: https://raw.githubusercontent.com/jiidata-science/jiidata-science.github.io/master/assets/images/posts/datascientists/DataScientists_title.png
excerpt: "Analysis of sought data science skills, using the Reed.co.uk API, with Pandas, Plotly & Scikit-Learn"
author_profile: true
author: Jez Irving
classes: wide

---

## Introduction & Purpose
Data Science has been popularly regarded as the "sexiest job of the 21st Century", according to the well-respected [Harvard business Review](https://hbr.org/2012/10/data-scientist-the-sexiest-job-of-the-21st-century).

The term 'Data Science', itself, has only recntly emerged, coined to reflect a new data specialisation that is broadly associated with making sense of vast stores of data. That is not to say that the process of exploring and analysing data hasn't been around for much longer. A Forbes article on ["A Very short History Of Data Science"](https://www.forbes.com/sites/gilpress/2013/05/28/a-very-short-history-of-data-science/#44f7dabc55cf) dates the origins of Data Science back to the 1960's, in association with John W. Tukey's publishings on Exploratory Data Analysis. For interests sake, Dr Tukey's (an American mathematician) was the creator of the popularly used box-plot

Since then, a combination of key figureheads & organisations, the Big Data revolution and a correlated growth in the need to draw insights from, make decisions from and, ultimately, moneytise large volumes of data, has brought us to where we are today - an information age ruled by Data Scientists. Sarcasm aside, data science seems to be 'very in' these days.

But what skills are really expected of a Data Scientists? And how do these compare with associated data professions, such as Data Analysis and Data Engineering. In pursuit of a clearer defition of what skills differentiate Data Scientists from other data jobs, in this article:

We use job descriptions available from a popular UK based job site (Reed.co.uk) to explore common and differentiating skills listed in job descriptions for Data Scientists, Analysts & Engineers
We finally develop a a simple classification model, with SciKit Learn, to understand how well we can classify a job description based on the required skills that it lists.

### Motivation

Over the last eight years I've been in data roles across a number of industries and companies. My role titles have ranged from Analyst, Consultant to Customer Scientist and Data Scientist (note: I've never explicity had a data engineering role). But how does a Data Scientist really understand the skills expected of them. Being a (relatively) consciencous data scientists, I'm always looking at new topics to learn and get my stuck into. Wouldn't it be great if I could focus my professional development towards what the market is really after (queue an idea for an article).

### What we'll be looking at

* Part 1 (of 5): Required Python Libraries
* Part 2 (of 5): Data collection - Using Reed.co.uk's **[JobSeeker API](https://www.reed.co.uk/developers/Jobseeker)** to retrieve job descriptions (with requests and pandas)
* Part 3 (of 5): Data Pre-Processing : Extracting Skills from Text Corpus
* Part 4 (of 5): Data Exploration - Popular Data Scientist skills & how they differ from other data professions (with regex and Plotly)
* Part 5 (of 5): Classifying a role based on required skills (quick classification model with [ScikitLearn](http://scikit-learn.org/stable/))

## Part 1 (of 5) : Required Python Libraries


``` python

# libraries for querying API
import requests
import json

# libraries data pre-processing (including regex)
import pandas as pd
import string
import nltk
from nltk.corpus import stopwords
import numpy as np
import re
from string import punctuation


# libraries (and configuration) for visualisation with Plotly
import matplotlib.pyplot as plt
from plotly.offline import download_plotlyjs, init_notebook_mode, plot, iplot
from plotly import __version__
import plotly.graph_objs as go
init_notebook_mode(connected=True)
%matplotlib inline

# libraries for classification model pipeline
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from sklearn.model_selection import train_test_split

```

## Part 2 (of 5): 

## Part 3 (of 5): 

## Part 4 (of 5): 

## Part 5 (of 5): 

## Outcomes




  
