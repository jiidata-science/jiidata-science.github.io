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

## Part 2 (of 5): Data collection - Using Reed.co.uk's **[JobSeeker API](https://www.reed.co.uk/developers/Jobseeker)** to retrieve job descriptions

Reed.co.uk have developed an API that allows partners to programmatically access jobs listed in their database. This service is available as an HTTP GET request, which can be parameterised to allow the user to perform specific searches (e.g. for specific roles, locations and salary expectations, to name a few). For more information refer to the [Reed.co.uk website](https://www.reed.co.uk/developers/Jobseeker).

I developed two simple functions that allowed me to return jobIds and Job descriptions for specific locations and role tiltes (i.e. data scientist, data analyst, data engineer); this is displayed in the code-block below.

getReedJobIDs : this function allows the user to request job IDs associated with a parameterised search. In my case, I used the keywords parameter and the locationName parameter to specify the role title (e.g. 'Data+Scientist') and the city name (e.g. london), respectively. This function returns a list with jobids associated with the query.

getJobDescription : this function allows the user to request the job descriptions for specified jobIds (retrieved using the getReedjobIDs function). This function returns a list with jobids and job descriptions, within. The job descriptions were unstructured raw text.

Importantly, the API had a response limit of 100 jobs per request. I had to request a token, which was passed as metadata in the GET request - unique to me

``` python
def getReedJobIDs( jobName , city ):
    base_url_request = 'https://www.reed.co.uk/api/1.0/search?keywords={0}&locationName={1}&graduate=false'.format(jobName , city)
    r = requests.get(base_url_request, auth=('336c2000-49ff-4229-a3cc-04303cf82eab', 'pass'))
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
    return([job_ids, query_name, query_location])

def getJobDescription( jobId ):
    base_url_request = 'https://www.reed.co.uk/api/1.0/jobs/{0}'.format(jobId)
    r = requests.get(base_url_request, auth=('336c2000-49ff-4229-a3cc-04303cf82eab', 'pass'))
    convert_json = json.loads(r.text)
    desc_raw = convert_json['jobDescription']
    cleanr = re.compile('<.*?>')
    desc_clean = re.sub(cleanr, '', desc_raw)
    return( [ jobId , desc_clean ] )
```

``` python
# we loop through two cities and three data job titles to return corresponding job descriptions
cities = ['london', 'manchester', 'glasgow', 'birmingham', 'liverpool', 'leeds', 'Bristol']
jobNames = ['data+scientist', 'data+analyst', 'data+engineer']

# get job ids based on query terms (in API)
jobsidsList = []
for city in cities:
    for job in jobNames:
        add = getReedJobIDs( job , city ) # get job ids for each city
        jobsidsList.append( add )

# create a single pandas dataframe with all jobs and job descriptions
DataScientist = pd.DataFrame( { 'JobID': jobsidsList[0][0], 'QueryTitle': jobsidsList[0][1], 'jobLocation': jobsidsList[0][2]})
DataAnalyst   = pd.DataFrame( { 'JobID': jobsidsList[1][0], 'QueryTitle': jobsidsList[1][1], 'jobLocation': jobsidsList[1][2]})
DataEngineer  = pd.DataFrame( { 'JobID': jobsidsList[2][0], 'QueryTitle': jobsidsList[2][1], 'jobLocation': jobsidsList[2][2]})
AllJobIDs     = pd.concat( [DataAnalyst, DataEngineer, DataScientist], axis = 0 )
```

## Part 3 (of 5): Data Pre-Processing - Extracting Skills from Text Corpus

So far we've reviewed the data collection process. We have approximately 100 job descriptions for each data role, with a total of ~300.

After ensuring that the full corpus of job descriptions included an equal volume of job descriptions per role, we started exploring the prevalence of different (data role) skills across the text.

The object of this project was to identify the required skills most associated with each data profession and how these skills differ between roles. With this in mind, I needed to first construct a comprehensive list of skills that accounted for the three roles. For this I used combined two approaches:

* Creating a comprehensive list of skills based on my knowledge / internet searches

    I chose to approach identifying 'skills' within our text corpus by defining an exhaustive list (or as exhaustive as I had the patience for) that represented all skills (of interest) associated with our job roles.

    I created two pairs of lists; one pair that represent regex for hard skills and the other pair for soft skills. Within each pair, one list includes a set of regex used to find a specific skill within a text document (e.g. r'\balgorithm\w*' for algorithm or algorithms) and a second list that includes 'clean' names used to create a binary flag if a text document contains the corresponding regex (e.g. 'algorithm').

* Performing tf-IDF on the text corpus itself to identify (additional) skills

    This step required using the text corpus itself. The purpose for this step was to capture any missed skills that hadn't been found as part of approach one. Any new terms or n-grams found with this approach were added to the original list of regular expressions developed in approach one.

    For this step, I did not pre-process the text data (such as removing stopwords, stemming or lemmatising). If we looked at term frequency, only, then common words such as ('the', 'and', 'a', 'an', 'because') would likely dominate the most popular terms - these terms are popularly referred to as stopwords. tf-IDf, however, is reltaively robust to the prevalence of stopwords when assessing the presence of interesting terms within a text corpus, by effectively downweighting the term frequency by the prevalence of the term across all text documents.

At this point I started cleaning our text corpus, ready for analysis. As our job descriptions were free text, we needed to extract useful information. When processing a corpus of text one typically begins by 'cleaning' the data, which may include removing stopwords (reference) , removing special characters (reference) aswell as stemming and/or lemmitising (reference) - for more details, analyticsvidhya.com have a ['Comprehensive Guide to text processing'](https://www.analyticsvidhya.com/blog/2018/04/a-comprehensive-guide-to-understand-and-implement-text-classification-in-python/)

Now we have a balanaced dataset, with 96 job descriptions for each data role.

Before we began extracting skills from the job descriptions, I first removed the role titles from the text - the purpose of this step was to remove words that could result in target leakage. For example, for data engineers it is likely that the term "engineer" or "engineering" will appear in corresponding job descriptions as a required skill (some examples below taken from the text corpus).

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

## Part 4 (of 5): Extracting skills

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

### Matching regex

``` python
hardMatches = [] # empty list object that will contain matched skills per document

# for each text document search for each skill (based on regex list)
for idx, doc in enumerate(text_removeStopPunc):
    list_matchedSkills = []
    for regvalue , regname in zip(regx_hardSkills_list , regx_hardfeatName_list):
        regexSearch = re.findall(regvalue , doc , re.I) # check if skills (regex) matched document
        # if atleast one match add regname to list
        if len(regexSearch) > 0:
            list_matchedSkills.append(regname)
    hardMatches.append(list_matchedSkills)
        
print('Done')
list(hardMatches[:3])
```

    #0utput:
    #[
    # ['Phd', 'MachineLearning', 'R'],
    # ['Algorithms','SQL','Spark','MachineLearning','Matlab','Python','Statistics','Testing','R','MapReduce','Hive','BigData'],
    # ['SQL', 'Phd', 'MachineLearning', 'Python', 'R']
    # ]

``` python
# ========== STEP 01 : create empty matrix / array
n = len(text_removeStopPunc)    # num. of docs
m = len(regx_hardfeatName_list) # by total num. of skills
skillsMatrix = [0] * n
for i in range(n):
    skillsMatrix[i] = [0] * m

# ========== STEP 02 : flag skillsets in each doc (using matrix)
for idx, eachSkillSet in enumerate(hardMatches):
    for eachSkill in eachSkillSet:
        indices = regx_hardfeatName_list.index(eachSkill)
        skillsMatrix[idx][indices] = 1

# ========== STEP 03 : convert matrix / visualise as pandas dataframe
df_matchedSkills = pd.DataFrame(skillsMatrix, columns=full_skills)
df_matchedSkills['Label'] = list(df_jobsClean_balanced.QueryTitle) # append the role title to the dataset
df_matchedSkills[:5]
```

| index | Algorithms | AppEngine |  NoSQL | SQL | Excel | Spark | ... | Label |
| ----- |:-----:| :-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| 0 | 0 | 0 | 0 | 0 | 0 | 0 | ... | Data Scientist |
| 1 | 1 | 0 | 0 | 1 | 0 | 1 | ... | Data Scientist |
| 2 | 0 | 0 | 0 | 1 | 0 | 0 | ... | Data Scientist |
| 3 | 0 | 0 | 0 | 1 | 1 | 0 | ... | Data Scientist |
| 4 | 0 | 0 | 0 | 0 | 0 | 1 | ... | Data Scientist |


## Part 5 (of 5): 

## Outcomes




  
