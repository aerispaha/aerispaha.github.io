---
layout: post
title: Trends in headlines from major media companies
---

In a recent post, I outlined a process that aims to take snapshots of the news media landscape overtime. That's led me  and store the time series in a database. After a some effort, I've been able to start extracting and analyzing the data with Pandas.


Let's start by taking a look at an example document in the MongoDB database representing a snapshot of USA Today's RSS feed on June 12, 2017:

```javascript
{
  u'_id': ObjectId('593e13134a5ac4327d88619c'),
  u'datetime': datetime.datetime(2017, 6, 12, 4, 5, 32, 832000),
  u'source': u'usa_today',
  u'stories': [
    u"'Dear Evan Hansen' wins six Tony Awards, including best musical",
    u"Penguins have become NHL's newest dynasty",
    u'First lady Melania Trump, son Barron officially move into the White House',
    u'E3 2017: The 5 biggest reveals during the Xbox event',
    u'Penguins repeat as Stanley Cup champions',
    u'Homophobic slur aimed at U.S. goalkeeper Brad Guzan by Mexican fans at World Cup qualifier',
    u"Ben Platt's reaction is the best GIF of the Tonys Awards",
    u'Puerto Ricans parade in New York, back statehood',
    u'Delta ends theater company sponsorship over Trump look-alike killing scene',
    u'U.S. earns rare tie vs. Mexico in World Cup qualifier at Estadio Azteca'
  ]
}
```

Within the document, we see a list of stories (i.e. news headlines) alongside the source (USA Today) and the datetime when the headlines were observed. These snapshots are created every 15 minutes with data from the RSS feeds of 19 media organizations. At the time of writing, there are 29,925 of these snapshots in the database with headlines starting on June 12, 2017 (note: I lost all data between June 13 and July 5).

Recording periodic snapshots allows us to observe how the media's interest in a particular topic changes over time. For example, a particular issue that is featured in USA Today's RSS feed for 30 hours must be of particular interest or importance, compared to another story that turns up for only a few hours. And, with data gathered four times per hour, we can construct time series that gives insight into how our media discourse changes over time and in response to important events.


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
      <th>2017-06-12 05:00:00</th>
      <td>cnn</td>
      <td>Engine part rips off China Eastern Airlines jet</td>
    </tr>
    <tr>
      <th>2017-07-14 12:45:00</th>
      <td>upi</td>
      <td>India rejects China's offer to mediate Kashmir...</td>
    </tr>
    <tr>
      <th>2017-07-12 17:45:00</th>
      <td>fox</td>
      <td>KFC China releasing Colonel-themed smartphone</td>
    </tr>
    <tr>
      <th>2017-07-22 03:45:00</th>
      <td>abc</td>
      <td>WATCH:  Suspected deadly gas explosion at food...</td>
    </tr>
    <tr>
      <th>2017-07-18 01:30:00</th>
      <td>wsj_world</td>
      <td>Unable to Buy U.S. Military Drones, Allies Pla...</td>
    </tr>
  </tbody>
</table>

## !!! STILL DRAFTING THIS POST !!!

## Coverage per Hour

next steps:
* how many stories exist a given point of time related to a topic
* compare all headlines at a time, find most common words/topic throughout
* measure the proportion given to particular topics, compared to the rest
* measure proportion given by a particular news source compared to the rest, on a given topic
