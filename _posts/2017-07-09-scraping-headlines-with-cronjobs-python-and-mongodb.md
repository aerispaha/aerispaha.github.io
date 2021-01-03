---
layout: post
title: Scraping headlines with cron jobs, Python, and MongoDB
tags: [Tech, Politics]
---

I've been interested in measuring bias in media coverage for quite a while. The discourse before and after the 2016 election has forced a lot of us (definitely me) into an anxiety spiral, trying to keep up and maintain sanity under the weight of our 24-hour news culture. To make myself feel better, I recently set up a cron job on a server that pulls the RSS feeds of as many news sources as I could think of, and stores the headlines in a MongoDB database.

To set up the cron job, I edited my crontab on my Ubuntu server by typing `crontab -e`. In the crontab I added the following line:

```crontab
*/15 * * * * /home/news_agg/scrape_news.py >/dev/null 2>&1
```

This basically tells the server to run a script located at `/home/news_agg/scrape_news.py` every 15 minutes. The `>/dev/null 2>&1` part prevents the cron from writing a message to a log each time it runs.

The news scraping Python script that gets executed every 15 minutes is fairly straightforward. First, [feedparser](https://github.com/kurtmckee/feedparser) (for parsing RSS feeds), [PyMongo](https://api.mongodb.com/python/current/) (Python wrapper for workin with MongoDB), and the datetime packages are imported and a dictionary of RSS feeds is constructed:

```python
#!/usr/bin/env python
import feedparser
import datetime
import pymongo

#create a dictionary of rss feeds
feeds = dict(
    nyt = r'http://rss.nytimes.com/services/xml/rss/nyt/HomePage.xml',
    fox = r'http://feeds.foxnews.com/foxnews/most-popular',
    wsj_opinion = r'http://www.wsj.com/xml/rss/3_7041.xml',
    wsj_business = r'http://www.wsj.com/xml/rss/3_7014.xml',
    wsj_world = r'http://www.wsj.com/xml/rss/3_7085.xml',
    wapo_national = r'http://feeds.washingtonpost.com/rss/national',
    cnn = r'http://rss.cnn.com/rss/cnn_topstories.rss',
    cnn_us = r'http://rss.cnn.com/rss/cnn_us.rss',
    breitbart = r'http://feeds.feedburner.com/breitbart',
    cnbc = r'http://www.cnbc.com/id/100003114/device/rss/rss.html',
    abc = r'http://feeds.abcnews.com/abcnews/topstories',
    bbc = r'http://feeds.bbci.co.uk/news/rss.xml',
    wired = r'https://www.wired.com/feed/',
    upi = r'http://rss.upi.com/news/top_news.rss',
    reuters = r'http://feeds.reuters.com/reuters/topNews',
    usa_today = r'http://rssfeeds.usatoday.com/usatoday-NewsTopStories',
    ap = r'http://hosted2.ap.org/atom/APDEFAULT/3d281c11a96b4ad082fe88aa0db04305',
    npr = r'http://www.npr.org/rss/rss.php?id=1001',
    democracy_now = r'https://www.democracynow.org/democracynow.rss',
)
```

From there, we loop through each of the RSS urls, pull out the title and source of each article, and store everything in a temporary `data` list of dicts. That list then gets written to a MongoDB collection called `headlines` in a database called `news`:
```python
#grab the current time
dt = datetime.datetime.utcnow()

data = []
for feed, url in feeds.iteritems():

    rss_parsed = feedparser.parse(url)
    titles = [art['title'] for art in rss_parsed['items']]

    #create dict for each news source
    d = {
        'source':feed,
        'stories':titles,
        'datetime':dt
    }
    data.append(d)

# Access the 'headlines' collection in the 'news' database
client = pymongo.MongoClient()
collection = client.news.headlines

#insert the data
collection.insert_many(data)
```
With the crontab in action, this script is run every 15 minutes which means we have a periodic snap shot of all of the new source's RSS feeds. This effectively creates a times series of news headlines at a 15-minute time step.

This script was kicked off on June 12, 2017 (about 30 days before the day of this post). Since then, I've only scratched the surface with analysis. I've also realized that my Mongo document structure is probably pretty awkward, but, hey, it works. As an example, I set up a query to count the amount of times a particular topic was found in each  new source's RSS feed:

```javascript
db.headlines.aggregate([
  {$unwind:"$stories"},
  {$match:{stories:/Yemen/}},
  {$group:{
    _id:"$source",
    stories:{$addToSet:"$stories"},
    count:{$sum:1}
  }},
]).pretty()
```

Here, I've queried the data for headlines including "Yemen", which returned the following:
```javascript
{
	"_id" : "nyt",
	"stories" : [
		"Cholera Spreads as War and Poverty Batter Yemen"
	],
	"count" : 87
}
{
	"_id" : "democracy_now",
	"stories" : [
		"Cholera Death Toll Tops 859 in War-Torn Yemen as U.S.-Backed Saudi Assault Continues"
	],
	"count" : 48
}
{
	"_id" : "fox",
	"stories" : [
		"Yemenis rally in support for secession of country's south",
		"Naval coalition steps up patrols around Yemen after attacks"
	],
	"count" : 2
}
```

This shows that, between June 12 and July 9, 2017, our script found that the RSS feeds of the New York Times, Democracy Now! and Fox News included stories about the war in Yemen 87, 48, and 2 times, respectively. Assuming each RSS feed snap shot was taken at 15 minute intervals, this suggests Yemen coverage could be found on NYT's feed for almost 22 hours while lasting just 30 minutes on Fox's feed.

The same query also suggests that aggregate RSS-coverage of Barron and Melania Trump moving into the Whitehouse lasted more than 50 hours across each news source.
