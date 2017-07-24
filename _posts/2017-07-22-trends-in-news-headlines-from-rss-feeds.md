---
layout: post
title: Trends in headlines from major media companies
---

In a recent post, I outlined a process that aims to take snapshots of the news media landscape overtime. That's led me  and store the time series in a database. After a some effort, I've been able to start extracting and analyzing the data with Pandas.


Let's start by taking a look at an example document in the MongoDB database representing a snapshot of USA Today's RSS feed on June 12, 2017:

```javascript
{
  "_id": ObjectId("593e13134a5ac4327d88619c"),
  "datetime": datetime.datetime(2017, 6, 12, 4, 5, 32, 832000),
  "source": u"usa_today",
  "stories": [
    "'Dear Evan Hansen' wins six Tony Awards, including best musical",
    "Penguins have become NHL's newest dynasty",
    "First lady Melania Trump, son Barron officially move into the White House",
    "E3 2017: The 5 biggest reveals during the Xbox event",
    "Penguins repeat as Stanley Cup champions",
    "Homophobic slur aimed at U.S. goalkeeper Brad Guzan by Mexican fans at World Cup qualifier",
    "Ben Platt's reaction is the best GIF of the Tonys Awards",
    "Puerto Ricans parade in New York, back statehood",
    "Delta ends theater company sponsorship over Trump look-alike killing scene",
    "U.S. earns rare tie vs. Mexico in World Cup qualifier at Estadio Azteca"
  ]
}
```

Within the document, we see a list of stories (i.e. news headlines) alongside the source (USA Today) and the datetime when the headlines were observed. These snapshots are created every 15 minutes with data from the RSS feeds of 19 media organizations. At the time of writing, there are 29,925 of these snapshots in the database with headlines starting on June 12, 2017 (note: I lost all data between June 13 and July 5).

Recording periodic snapshots allows us to observe how the media's interest in a particular topic changes over time. For example, a topic that's discussed simultaneously in multiple articles by many media organizations must be especially important at a given point in time. Similarly, an issue that is featured throughout the RSS feeds for many days are likely more significant than stories that come and go within a few hours.

-- Rainfall intensity duration note here --

And, with data gathered four times per hour, we can construct time series that gives insight into how our media discourse changes over time and in response to important events.


## Querying Headlines with PyMongo and Pandas
In order to interact with the data, we construct a function, `query_rss_stories()` that wraps up the process of connecting to MongoDB, querying using the aggregation framework and regex, and organizing the results in a Pandas Dataframe:

```python
def query_rss_stories(regex):

    #connect to db
    client = MongoClient()
    news = client.news
    headlines = news.headlines

    #query the database, create dataframe
    pipeline = [
        {"$unwind":"$stories"},
        {"$match":{"stories":{'$regex':regex}}},
        {"$project":{"_id":0, "datetime":1, "stories":1, "source":1}}
    ]

    #unpack the cursor into a Dataframe
    cursor = headlines.aggregate(pipeline)
    df = pd.DataFrame([i for i in cursor])

    #round the rss query datetimes to 15mins
    df.datetime = df.datetime.dt.round('15min')
    raw_stories = df.set_index(['datetime'])

    return raw_stories
```

As an example, we'll search for stories related to China:
```python
stories = query_rss_stories('China')
stories.sample(n=5)
```
<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>source</th>
      <th>stories</th>
    </tr>
    <tr>
      <th>datetime</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2017-07-20 14:00:00</th>
      <td>breitbart</td>
      <td>Report: U.S. Bans Prompt American Allies to Bu...</td>
    </tr>
    <tr>
      <th>2017-07-14 05:45:00</th>
      <td>upi</td>
      <td>India rejects China's offer to mediate Kashmir...</td>
    </tr>
    <tr>
      <th>2017-07-13 00:15:00</th>
      <td>wsj_opinion</td>
      <td>How to Squeeze China</td>
    </tr>
    <tr>
      <th>2017-07-07 19:00:00</th>
      <td>abc</td>
      <td>US bombers fly over East and South China Seas</td>
    </tr>
    <tr>
      <th>2017-07-12 19:15:00</th>
      <td>reuters</td>
      <td>China dissident Liu's condition critical, brea...</td>
    </tr>
  </tbody>
</table>
</div>

The `stories` dataframe contains all headlines related to China from all of the new sources starting around July 6. For every hour that a headline is live on a particular RSS feed, four entries will be found in the `stories` dataframe (since the snapshots are taken at 15-minute intervals).

Next, we unstack the data and count the number of China-related headlines found at each time step:

```python
def create_rss_timeseries(df, freq='1h'):
    #unstack sources and create headline_count time series
    ts = df.assign(headline_count = 1)
    ts = ts.groupby(['datetime', 'source']).sum()
    ts = ts.unstack(level=-1)
    ts.columns = ts.columns.get_level_values(1)

    #set time step
    ts = ts.assign(datetime = ts.index)
    ts = ts.groupby(pd.Grouper(key='datetime', freq=freq)).mean()

    return ts

#daily time series of China headline counts
ts = create_rss_timeseries(stories, freq='1d')
```

Now the data is ready to be plotted:
![China-Headlines-Count]({{site.url}}/assets/img/china-headlines-count.png)



## Coverage per Hour

next steps:
* how many stories exist a given point of time related to a topic
* compare all headlines at a time, find most common words/topic throughout
* measure the proportion given to particular topics, compared to the rest
* measure proportion given by a particular news source compared to the rest, on a given topic
