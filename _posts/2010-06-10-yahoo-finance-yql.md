---
layout: post
title: Retrieving Yahoo! Finance Data using YQL 
summary: Getting stock information programmatically through Yahoo! Finance
date: 2010-06-10
tags: python YQL
---

Script available [here](http://github.com/gurch101/StockScraper).

YQL is a query language that lets you retrieve data from across the web using an SQL-like syntax. By making a simple REST query using a standardized syntax, you can receive XML- or JSON-formatted data from a bunch of web service APIs. You can play around with YQL [here](http://developer.yahoo.com/yql/console/).

I wrote a YQL wrapper in python which queries the Yahoo! Finance open data tables for financial data. Using this script, you can get current and historical financial info, news feeds, and options data for any ticker symbol.

### Usage

{% highlight python %}
import stockretriever as stocks

# Get current stock information - returns most of 
# the information on a typical Yahoo! Finance stock page
info = stocks.get_current_info(["YHOO","AAPL","GOOG","MSFT"])

# Get current stock news - returns the RSS feed for
# the given ticker in JSON format
news = stocks.get_news_feed('YHOO')

# Get historical prices - returns all historical
# open/low/high/close/volumn for thie given ticker
news = stocks.get_historical_info('YHOO')

# Get options data
options = stocks.get_options_data('YHOO')
{% endhighlight %}
