---
layout: post
title: Syntactic Clustering of News Headlines 
summary: grouping together news articles by subject using tf-idf weighting
date: 2010-07-12
tags: java
---

I’ve been using Google Reader to keep up with around 75 RSS feed subscriptions. Unfortunately there’s a lot of overlap between articles, especially articles from traditional news outlets. Google News groups similar articles so I wanted to see if I could replicate this behaviour for my RSS feeds (Yea, I know that I could just grab the RSS feed straight from google news or any other aggregator but where’s the fun in that ;-) ).

Here, I look at the syntactic similarity between RSS items from 31 news outlets in an attempt to categorize the news by subject.

### Fetching RSS Feeds

To get my feeds, I used the YQL rss.multi.list data table. which parses any RSS/atom feed and standardizes it into XML, HTML, or JSON. I retrieved my feeds in XML then used XPath to parse individual RSS feed items, stripped any HTML, and built RSSItem objects that I could easily manipulate programmatically.

### Preprocessing

For each RSSItem object, I built a length-normalized map of word frequencies for the 20 most ‘important’ words. Instead of finding the frequency of words as-is, I stemmed each word with a [Porter stemmer](http://tartarus.org/~martin/PorterStemmer/) which reduces derived/inflected words to their root form. I then evaluate the ‘importance’ of each word – if the word occurs frequently in a document but relatively infrequently in the entire corpus, then its important; if the word occurs frequently across many documents, its unimportant (this is the general premise of [tf-idf weighting](http://en.wikipedia.org/wiki/Tf-idf)).

### Clustering

To cluster feeds together, I take the intersection of the most important words for each feed. If each document shares at least 4 ‘important’ words (works out to ~20-40% of ‘important’ words shared), they’re considered to be on the same topic. Here’s the code that does the clustering:

{% highlight java %}
public static List<DocumentCluster> cluster(List<DocumentCluster> clusters){
    return cluster(clusters, MIN_WORDS_MATCH, 5);
}

private static List<DocumentCluster> cluster(List<DocumentCluster> clusters, int minMatch, int numIterations){
    if(numIterations < 0){
        return clusters;
    }
    int numClustered = 0;
    for(int i = 0; i < clusters.size(); i++){
        DocumentCluster c1 = clusters.get(i);
        if(c1.size() > 0){
            for(int j = i + 1; j < clusters.size(); j++){
                DocumentCluster c2 = clusters.get(j);
                if(c2.size() > 0){
                    int numShared = getNumSharedWords(c1,c2);
                    if(numShared > minMatch){
                        c1.merge(c2);
                        c2.removeDocuments();
                        numClustered++;
                    }
                }
            }
        }
    }
    
    ArrayList<DocumentCluster> clusterList = new ArrayList<DocumentCluster>();
    for(int i = 0; i < clusters.size(); i++){
        if(clusters.get(i).size()>0)
            clusterList.add(clusters.get(i));
    }
    
    //no clusters formed on this iteration
    if(numClustered == 0){
        return clusterList;
    }
    return cluster(clusterList, minMatch, numIterations -1);
}

public static int getNumSharedWords(DocumentCluster c1, DocumentCluster c2){
    Set<WordStem> topWords2 = new HashSet<WordStem>(c2.getTopWords());
    Set<WordStem> topWords1 = new HashSet<WordStem>(c1.getTopWords());
    topWords1.retainAll(topWords2);
    return topWords1.size();
}
{% endhighlight %}

### Analysis

I pulled news from 31 sources focused primarily on top stories, world, and national news. From the 526 RSS news items, 306 clusters were formed with at least 2 news stories each. Of the 306, 21 were incorrectly clustered (stories were unrelated to those in the cluster). Of the remaining 220 stories which weren’t part of a cluster,  89 should have been in a cluster. Not being one to give up an opportunity to draw a graph, here’s the above info represented visually :-)

![Document Classification Breakdown](/images/document_classification_breakdown.png)

True positives are news items that were correctly clustered. False positives are news items that were incorrectly clustered. True negatives were correctly identified as a unique story (not part of a cluster). False negatives were incorrectly identified to be unique or part of a cluster that should have been merged with another cluster.

The majority of true negatives were lifestyle/editorial style stories. False negatives were mostly articles with quotes/anecdotes in the headline. I tried to decrease the number of false negatives by lowering the number of words that need to be shared but this inevitably led to a decrease in specificity.

Although improvements could probably be made by looking at semantic similarity or by considering a larger body of text (the average rss item contained about 50 words of which only ~20 were ‘important’), syntax-based comparisons provide results that I’d consider ‘good enough’.
