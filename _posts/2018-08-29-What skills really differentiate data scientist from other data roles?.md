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

In this article we look to identify what skills really differentatiate Data Scientists from other data roles - namely, Data Analysts and Data Engineers. We use an API provided by a popular UK-based job site, Reed.co.uk, to collect listed jobs and their descriptions, for Data Science, Analyst & Engineering roles. Through processing this text data, with regular expressions, we extract, compare and contrast the skills that the job market seeks for each role. If you're interested to understand how the job market differentiates between these roles OR you're a budding data scientist and want to develop relevant skills, this article will provide some useful info...enjoy!

### Some brief context...

Whilst *Data Science* is regarded as the *"sexiest job of the 21st Century"* (cue valid source : [Harvard business Review](https://hbr.org/2012/10/data-scientist-the-sexiest-job-of-the-21st-century)) the term has only recently emerged, coined to reflect a new data specialisation that is broadly associated with making sense of vast stores of data. That is not to say that the process of exploring and analysing data hasn't been around for much longer. A Forbes article on ["A Very short History Of Data Science"](https://www.forbes.com/sites/gilpress/2013/05/28/a-very-short-history-of-data-science/#44f7dabc55cf) dates the origins of Data Science back to the 1960's, in association with John W. Tukey's publishings on Exploratory Data Analysis (note: for interests sake, Dr Tukey (an American mathematician) was the creator of the popularly used box-plot).

Since then, a combination of key figureheads & organisations, the Big Data revolution and a correlated growth in the need to draw insights from, make decisions from and, ultimately, moneytise large volumes of data, has brought us to where we are today - an information age ruled by Data Scientists. Sarcasm aside, data science seems to be 'very in' these days.

*But what skills are really expected of a Data Scientist?; and how do these compare with associated data professions, such as Data Analysis and Data Engineering?* In pursuit of some answers to these questions, in this article, we use job descriptions available from a popular UK based job site (Reed.co.uk) to explore common and differentiating skills listed in job descriptions for Data Scientists, Analysts & Engineers. We finally develop a a simple classification model, with SciKit Learn, to understand how well we can classify a job description based on the required skills that it lists.

### Motivation

Over the last eight years I've been in data roles across a number of industries and companies. My role titles have ranged from Analyst, Consultant to Customer Scientist and Data Scientist (note: I've never explicity had a data engineering role). But how does a Data Scientist really understand the skills expected of them. Being a (relatively) consciencous data scientists, I'm always looking at new topics to my teet into. Wouldn't it be great if I could focus my professional development towards what the market is really after (cue obvious idea for an article).

### What we'll be looking at

* Part 1 (of 5): Required Python Libraries (for project)
* Part 2 (of 5): Data collection - Using Reed.co.uk's **[JobSeeker API](https://www.reed.co.uk/developers/Jobseeker)** to retrieve job descriptions (with requests and pandas)
* Part 3 (of 5): Data Pre-Processing - Extracting Skills from Text Corpus
* Part 4 (of 5): Data Exploration - Popular Data Scientist skills & how they differ from other data professions (with regex and Plotly)
* Part 5 (of 5): Classifying a role based on required skills (quick classification model with [ScikitLearn](http://scikit-learn.org/stable/))

## Part 1 (of 5) : Required Python Libraries

This step is pretty trivial so little explanation is needed in addition to the commented code-block below. It is perhaps useful to note, however, that in order to use Plotly, within a jupyter notebook, I had to use the plotly.offline configuration and specify init_notebook_mode(connected=True) .


``` python
# libraries for querying API
import requests
import json

# libraries data pre-processing (including regex)
import pandas as pd
import string
import nltk
import numpy as np

import re # for identifying skills, in text, using regular expressions
from nltk.corpus import stopwords # for removal of stopwords from corpus
from string import punctuation # for removal of special characters / punctuation from corpus

# libraries for classification model pipeline
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from sklearn.model_selection import train_test_split

# libraries (and configuration) for visualisation with Plotly
import matplotlib.pyplot as plt
from plotly.offline import download_plotlyjs, init_notebook_mode, plot, iplot
from plotly import __version__
import plotly.graph_objs as go
init_notebook_mode(connected=True)
%matplotlib inline

```

## Part 2 (of 5): Data collection - Using Reed.co.uk's **[JobSeeker API](https://www.reed.co.uk/developers/Jobseeker)** to retrieve job descriptions

Reed.co.uk have developed a **FREE API** that allows partners to programmatically access jobs listed in their database. This service is available as an HTTP GET request, which can be parameterised to allow the user to perform specific searches (e.g. from specific roles, locations and salary expectations, to name a few). For more information refer to the [Reed.co.uk website](https://www.reed.co.uk/developers/Jobseeker). To use this service all you need to do is sign up as a 'partner' by providing a few basic details to retrieve a personal authorisation token, which is passed as metadata when calling the API.

I developed two simple functions that allowed me to return jobIds and Job descriptions for specific locations (e.g. london, manchester, glasgow ...) and role tiltes (i.e. data scientist, data analyst, data engineer), displayed in the code-block below. To summarise their uses:

* **getReedJobIDs()** : this function allows the user to request jobIDs associated with a parameterised search. In my case, I used the *keywords* parameter and the *locationName* parameter to specify the role title (e.g. 'data+scientist') and the city name (e.g. 'london'), respectively. This function returns a list with jobids associated with your query parameters.

* **getJobDescription()** : this function allows the user to request the job descriptions for specified jobIds (first retrieved using the getReedjobIDs() function, above). This function returns a list with jobids and job descriptions, within. The job descriptions were unstructured raw text.

Importantly, Reed.co.uk's API has a [response limit of 100 jobs per keyword](https://www.reed.co.uk/developers/Jobseeker), as specified on their website. Therefore, we're expected just 100 unique job descriptions per role.

``` python
def getReedJobIDs( jobName , city ):
    base_url_request = 'https://www.reed.co.uk/api/1.0/search?keywords={0}&locationName={1}&graduate=false'.format(jobName , city)
    r = requests.get(base_url_request, auth=('<enter-authorisation-token-here', 'pass')) # this is where you put the authorisation token
    convert_json = json.loads(r.text)
    job_ids = []
    query_name = []
    query_location = []
    for each_result in convert_json['results']:
        for key, value in each_result.items():
            if key == 'jobId':
                job_ids.append(value) 
                query_name.append(jobName)
                query_location.append(city)       
    return( job_ids )

def getJobDescription( jobId ):
    base_url_request = 'https://www.reed.co.uk/api/1.0/jobs/{0}'.format(jobId)
    r = requests.get(base_url_request, auth=('336c2000-49ff-4229-a3cc-04303cf82eab', 'pass'))
    convert_json = json.loads(r.text)
    desc_raw = convert_json['jobDescription']
    cleanr = re.compile('<.*?>')
    desc_clean = re.sub(cleanr, '', desc_raw)
    return( [ jobId , desc_clean ] )
```

We then used these functions to collect our data. Now typically this would be the point I'd insert a code block that shows what/how I performed this task - however, as this was over 80 lines of code I decided to show the output dataset and provide a [link](www.google.com) to the full code set.

The method that I applied to collect job descriptions was as follows:

1. Used the **getReedJobIDs()** function to retrieve job IDs associated with the following query paramters:
    * keywords: ['data+scientist' , 'data+engineer' , 'data+analyst']
    * locations: ['london' ,'birmingham','glasgow', 'birmingham', 'liverpool', 'leeds']
2. Deduplicated job IDs
3. Balanced the returned dataset to ensure there was an equal number job descriptions for each location (i.e. remove any inherent location-based bias)
4. Used the **getJobDescription()** function to retrieve job descriptions (raw unstructured text) for each jobID
5. Converted all returned data into Pandas dataframe and exported to .csv

So here it is...the output dataset contains the JobID, jobLocation, jobTitle and the jobDesc (i.e. job description). The final dataset had 375 records (125 x job descriptions per role), with ~78% referring to London based opportunities.

|JobID|jobLocation|jobTitle|JobDesc|
|---|---|---|---|
| 35672272 |   london  |   data+engineer   |   Data Engineer (Python & noSQL) An opportunity f... |
| 35897057 |   london  |   data+scientist  |   Data Scientist London We're looking for someone... |
| 35738489 |   london  |   data+scientist  |   Python Developer/Data Scientist - 6 Months - ... |
| 35788033 |   london  |   data+scientist  |   Data Scientist- Data Science Jobs, Statistica... |
| 35853388 |   london  |   data+scientist  |   Lead Data Scientist London &#163;100,000 ... |
| 35940926 |   london  |   data+analyst    |   Data Analyst - CRM team looking for a a perm... |


## Part 3 (of 5): Data Pre-Processing - Creating a list of skills & data cleaning

So far we've reviewed the data collection process. At this point I had 125 x job descriptions for each data role, with a total of 375.

### 3a. Removing the Job Title from the job description

Before we began flagging skills from the job descriptions, I first removed the role titles from the text - the purpose of this step was to remove words that could result in target leakage. For example, for data engineers it is likely that the term "engineer" or "engineering" will appear in corresponding job descriptions as a required skill (some examples below taken from the text corpus).

> "The business requires <mark>creative engineering</mark> balanced with high quality and customer focus"

> "You will enter the team at an early stage, deploying and monitoring their unique system and providing <mark>engineering support</mark> to customers."

> "Provide direct support to users about their deployment and <mark>data engineering needs</mark>"

Nevertheless, the terms "engineer" and "engineering" will also be used to refer to the relevant job title, such as <mark>Data Engineer</mark> or <mark>Lead Engineer</mark>. Remember, the whole purpose here is to extract skills that corresponding with specific role titles and therefore we removed such references from the text to avoid over-representing related skills.

``` python
# specificy regex terms for each job title (+ variance of)
leakage_regex = [  r'\s?lead\s+data\s+engineer\w?' # (lead data engineer(s))  
                 , r'\s?senior\s+data\s+engineer\w?' # (senior data engineer(s)) 
                 , r'\s?data engineer\w?' # (data engineer | data engineers)
                 , r'\sengineer\w?' # (engineer(s))             
                 , r'\s?lead\s+data\s+scient\w+' # (lead data scientist(s)) 
                 , r'\s?senior\s+data\s+scient\w+' # (senior data scientist(s))
                 , r'\s?data scient\w+'  # (data scientist(s))
                 , r'\bscientist\w?' # (scientist(s))
                 , r'\s?lead\s+data\s+analyst\w?' # (lead data engineer(s)) 
                 , r'\s?senior\s+data\s+analyst\w?' # (senior data analyst(s)) 
                 , r'\s?data analyst\w?' # (data analyst(s))
                 , r'\banalyst\w?'
                   ]

# function that will take a input text string and remove regex expressions
def removeJobTitles(txt_string, regex_list):
    try:
        outtxt = txt_string # initiate output as the input string
        for idx, regx in enumerate(regex_list):
            p = re.compile(regx, re.I)
            outtxt = p.sub("", outtxt)    
    except Exception as e:
        print(e.args)
    else:
        return outtxt

# call function
text_noJobTitles = list( [removeJobTitles(string , leakage_regex) for string in list(df_jobsClean_balanced.jobdescription)])
```

### 3b. Removing stopwords & punctuation

- After this I started to we clean the text and then extract the skills that are associated with these data roles

``` python
text_removeStopPunc = []
allstopwords = stopwords.words('english')+list(punctuation)

for descrip in text_noJobTitles:    
    descrip = [word.lower() for word in descrip.split() if word.lower() not in allstopwords]
    descrip = ' '.join(descrip)
    text_removeStopPunc.append(descrip)
    del descrip
```







## Part 4 (of 5): Creating a skills matrix

### 4a. Creating a list of skills using regex

Next I started to explore the presence of different required skills across job descriptions. The objective of this project was to identify the required skills most associated with each data profession and how these skills differ between roles. With this in mind, I first constructed a comprehensive list of skills that accounted for the three roles. To create this list I combined two approaches:

* **(1) Creating a comprehensive list of skills based on my knowledge / internet searches**

    This approach involved defining an exhaustive list (or as exhaustive as I had the patience for) that represented all skills (of interest) associated with our job roles.

    I created two pairs of lists; one pair that represent regex for hard skills and the other pair for soft skills. Within each pair, one list includes a set of regex used to find a specific skill within a text document (e.g. r'\balgorithm\w*' for algorithm or algorithms) and a second list that includes 'clean' names used to create a binary flag if a text document contains the corresponding regex (e.g. 'algorithm').

* **(2) Performing tf-IDF on the text corpus itself to identify (additional/missed) skills**

    This step required using the text corpus itself. The purpose for this step was to identify any missed skills that hadn't been found as part of approach one. Any newly identified terms or n-grams were added to the original list.

    Before performing tf-IDf I removed stopwords - these are words such as ('the', 'and', 'a', 'an', 'because') that are commonly used but do not carry any importance. We used ```nltk.stopwords``` as out reference list of stopwords.

For a more detailed overview of text processing, with Python, Analyticsvidhya.com have created a ['Comprehensive Guide to text processing'](https://www.analyticsvidhya.com/blog/2018/04/a-comprehensive-guide-to-understand-and-implement-text-classification-in-python/)

- Create a list of skills that should cover the three data professions
- Explore the text corpus itself using tfIDf to identify other skills that were left out of the skill set
- the skills were represented as regex terms given slight variations

```
regx_hardSkills_list = [r'algorithm\w*' , r'\bapp\seng\w*\b' , r'\w*nosql' , r'\bsql\b', r'\bexcel\b'
            , r'spark' , r'phd', r'\bmetric\w?\b', r'\w*aws\w*\b' , r'java' , r'predict\w+\b'
            , r'\bpipeline\w+\b', r'\bmining\b', r'\bjavascript\b', r'js\b',  r'decision\s*tree\w+\b'
            , r'\becl\b' , r'hadoop' , r'hbase\w*\b', r'\balgebra\w*\b', r'machine\s*l\w+\b'
            , r'\w*math\w*\b' , r'\w*matlab\w*\b' ,r'\bcalculus\b', r'\bperl\b'
            , r'\w*powerpoint\w*\b', r'\w*visualis\w*\b' , r'\bprogramming\b', r'\w*python\w*\b' , r'scikit\w*', r'\w*pandas\w*\b'
            , r'dply\w*\b' , r'\w*report\w*\b' , r'\bsas\w*\b', r'\bscripting\b' , r'statistic\w*\b'
            , r'kpi\w?\b' , r'tableau' , r'\badwords\w?\b' , r'\btest\w*\b' , r'\bhypothes\w+\b'
            , r'\bhtml\b' , r'\w*pivot\w?\b' , r'\bdesign\w*\b' , r'\bcampaign\w*\b' , r'\w*google\s?analytics\w*\b'
            , r'\w*agile\w*\b' , r'\w*scrum\w*\b' , r'\bcloudera\b' , r'\w*cloud\b' , r'json\w*' , r'\bapi\w?\b'
            , r'\bsecurity\b' , r'\br\b' , r'dashboard' , r'qlikview' , r'spss'
            , r'\bbi\b|\bbusiness\s?int\w*\b' , r'\bruby|\sruby' , r'\w*scala\b' , r'php' , r'vba|visual\s?bas\w*'
            , r'mongo' , r'cassandra' , r'\w*map\s?reduce\w*' , r'pig'
            , r'\w*hive\w*' , r'adobe' , r'nlp' , r'big\s?data' , r'msc', r'\bc\b|c\#' , r'\w*anomalie\w*\b'
            , r'redshift' , r'integrat\w*', r'segment\w*' , r'gcp' , r'azure', r'data\s?management'
            , r'\w*deep\s?learning' , r'data\s+modelling' , r'data\s?quality' , r'computer\s?vision'  ]

regx_hardfeatName_list = ['Algorithms' , 'AppeEngine' , 'NoSQL' , 'SQL' , 'Excel' , 'Spark' , 'Phd' , 'Metrics' , 'AWS' 
            , 'Java' , 'Predictions', 'Pipelines' , 'DataMining', 'Javascript' , 'Javascript' , 'Decision Trees'
            , 'ECL' , 'Hadoop', 'HBase', 'Algebra', 'MachineLearning'
            , 'Mathematics', 'Matlab', 'Calculus' , 'Perl'
            , 'Powerpoint', 'Visualisation', 'Programming', 'Python', 'ScikitLearn', 'Pandas'
            , 'dplyR', 'ReportWriting', 'SAS', 'Scripting', 'Statistics'
            , 'KPIs' , 'Tableau' , 'AdWords', 'Testing', 'HypothesisTesting'
            , 'HTML', 'PivotTables', 'Design', 'Campaigns', 'GoogleAnalytics'
            , 'Agile', 'Scrum', 'Cloudera', 'Cloud', 'JSON', 'APIs'
            , 'Security' , 'R', 'Dashboards', 'Qlikview', 'SPSS'
            , 'BI', 'Ruby' , 'Scala', 'php', 'VBA'
            , 'Mongo', 'Cassandra', 'MapReduce', 'Pig'
            , 'Hive', 'Adobe' , 'NLP', 'BigData', 'MSc', 'C' , 'AnomalyDetection'
            , 'RedShift', 'Integration', 'Segmentation', 'GCP', 'Azure', 'DataManagement'
            , 'DeepLearning' , 'DataModelling' , 'DataQuality' , 'ComputeVision']
```

### 4b. Creating a sparse matrix of skills in each document

``` python
# create an empty m x n matrix (m = skills , n = job descriptions)
n = len(text_removeStopPunc)    # num. of docs
m = len(regx_hardfeatName_list) # by total num. of skills
skillsMatrix = [0] * n
for i in range(n):
    skillsMatrix[i] = [0] * m
    
# flag the skills in each document    
for idx, doc in enumerate(text_removeStopPunc):
    for regvalue , regname in zip(regx_hardSkills_list , regx_hardfeatName_list):
        regexSearch = re.findall(regvalue , doc , re.I) # check if skill is present
        # if atleast one match add regname to list
        if len(regexSearch) > 0:            
            indices = regx_hardfeatName_list.index(regname)
            skillsMatrix[idx][indices] = 1

# convert sparse/dummy matrix to pandas dataframe
df_matchedSkills = pd.DataFrame(skillsMatrix , columns = regx_hardfeatName_list)
df_matchedSkills['Label'] = list(df_jobsClean.jobTitle) # append the role as label
df_matchedSkills[:5] # show top results
```

| index | Algorithms | AppEngine |  NoSQL | SQL | Excel | Spark | ... | Label |
| ----- |:-----:| :-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| 0 | 0 | 0 | 0 | 0 | 0 | 0 | ... | Data Scientist |
| 1 | 1 | 0 | 0 | 1 | 0 | 1 | ... | Data Scientist |
| 2 | 0 | 0 | 0 | 1 | 0 | 0 | ... | Data Scientist |
| 3 | 0 | 0 | 0 | 1 | 1 | 0 | ... | Data Scientist |
| 4 | 0 | 0 | 0 | 0 | 0 | 1 | ... | Data Scientist |

## Part 5 (of 5): Analysis

**A couple of general points**:

- We searched the text corpus for a total of 81 x skills. Five skills had no matches for across roles : ['AppEngine', 'Algebra', 'Calculus', 'Perl', 'dplyR']
- The total number of skills present for each role was: Data Science (67 skills), Data Engineer (66 skills) and Data Analyst (55 skills)

**A look at the most in-demand skills for Data Scientists**:

This chart below the proportion of job descriptions that contain a particular skill, for each of the data roles (note: the blue bar represents data scientists as they were the main focus of my analysis. The purple scatter/dots represent data engineers and the green, data analysts). I've ordered the chart by prevalence of each skill across data science job descriptions.

<iframe width="1000" height="550" frameborder="0" scrolling="no" src="//plot.ly/~jii-datascience/14.embed"></iframe>

- Python *( **Scientist : 99%** , Engineer : 78% , Analyst : 25% )*
- Machine Learning *( **Scientist : 89%** , Engineer : 31% , Analyst : 8% )*
- R *( **Scientist : 71%** , Engineer : 11% , Analyst : 13% )*
- SQL *( **Scientist : 63%** , Engineer : 54% , Analyst : 52% )*, and 
- Statistics *( **Scientist : 53%** , Engineer : 7% , Analyst : 24% )*

Additionally, a higher proportion of data science jobs require these five skills than other data roles. It's quite clear that a budding data scientist should focus on upskilling in each of these areas. Whilst each person has their own coding language preference, I think it's useful for a data scientist to have a working appreciation for both R and Python as they may each be (slighltly) better suited to different tasks.

The above chart also illustrates the skills that are far more required for engineering (e.g. Spark and Hadoop) and analyst (e.g. Report Writing and BI) roles, than for data science. **Feel free to explore the chart yourself, but I think we can better visualise the skills that are more specific to data scientists and perhaps which are common across all three.**

**Comparing skills more specific to Data Scientists that other roles**:

<iframe width="1000" height="550" frameborder="0" scrolling="no" src="//plot.ly/~jii-datascience/20.embed"></iframe>

## Outcomes




  
