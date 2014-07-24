---
layout: post
title: "Combining pages of JSON data with jsonlite and plyr"
category: posts
description: "A recording of the useR! 2014 talk about OpenCPU is now available on Youtube. This talk gives a brief (20 minute) motivation and introduction to some of the high level concepts of the OpenCPU system."
cover: "containers.jpg"
---

The [jsonlite](http://cran.r-project.org/web/packages/jsonlite/index.html) package is a `JSON` parser/generator for R which is optimized for pipelines and web APIs. It is used by the OpenCPU system and many other packages to get data in and out of R using the `JSON` format.

## A bidirectional mapping

One of the main strenghts of `jsonlite` is that it implements a bidirectional [mapping](http://arxiv.org/abs/1403.2805) between data frames and `JSON`. Thereby it can convert nested collections of `JSON` records, as they often appear on the web, immediately into the appropriate R structures, without complicated manual data munging by the user. For example, if a journalist wants to grab some data from ProPublica, she can simply use something like:

{% highlight r %}
library(jsonlite)
mydata <- fromJSON("http://projects.propublica.org/forensics/geos.json")
View(mydata$geo)
{% endhighlight %}

Here, the `mydata$geo` object is a data frame which can be used directly for modeling or visualization, without the need for advanced data minipulation skills.

## Paging with jsonlite and plyr

A question that comes up frequently is how to combine pages of data. Most web APIs limit the amount of data that can be retrieved per request. If the client needs more data than what can fits in a single request, it needs to break down the data into multiple requests that each retrieve a fragment (page) of data, not unlike pages in a book. In practice this is often implemented using a `page` parameter in the API. Below an example from the [ProPublica Nonprofit Explorer API](http://projects.propublica.org/nonprofits/api) where we retrieve the first 3 pages of tax-exempt organizations in the USA, ordered by revenue:

{% highlight r %}
baseurl <- "http://projects.propublica.org/nonprofits/api/v1/search.json?order=revenue&sort_order=desc"
mydata0 <- fromJSON(paste0(baseurl, "&page=0"))
mydata1 <- fromJSON(paste0(baseurl, "&page=1"))
mydata2 <- fromJSON(paste0(baseurl, "&page=2"))

#The actual data is in the filings element
print(mydata0$filings)
print(mydata0$filings$organization)
{% endhighlight %}

To analyze or visualize these data, we need to combine the pages into a single dataset. This is best done using `rbind.fill` from the `plyr` package. However because `rbind.fill` does not support nested data frames, we need to flatten the `JSON` data by passing the `flatten = TRUE` argument to `fromJSON`.

{% highlight r %}
#Note flatten=TRUE requires jsonlite => 0.9.9
baseurl <- "http://projects.propublica.org/nonprofits/api/v1/search.json?order=revenue&sort_order=desc"
mydata0 <- fromJSON(paste0(baseurl, "&page=0"), flatten = TRUE)
mydata1 <- fromJSON(paste0(baseurl, "&page=1"), flatten = TRUE)
mydata2 <- fromJSON(paste0(baseurl, "&page=2"), flatten = TRUE)

#Combine data pages
library(plyr)
filings <- rbind.fill(mydata0$filings, mydata1$filings, mydata2$filings)

#Check output
colnames(filings)
nrow(filings)
{% endhighlight %}

## Automatically combining many pages

We can write a simple loop that automatically downloads and combines many pages. For example to retrieve the first 20 pages with non-profits from the example above:

{% highlight r %}
#requires jsonlite >= 0.9.9
library(jsonlite)

#store all pages in a list first
baseurl <- "http://projects.propublica.org/nonprofits/api/v1/search.json?order=revenue&sort_order=desc"
pages <- list()
for(i in 0:20){
  mydata <- fromJSON(paste0(baseurl, "page=", i), flatten=TRUE)
  message("Retrieving page ", i)
  pages[[i+1]] <- mydata$filings
}

#combine all into one 
library(plyr)
filings <- rbind.fill(pages)

#check output
nrow(filings)
colnames(filings)
{% endhighlight %}

From here, our journalist can go straight to analyzing the data without any further tedious, complicated and time consuming data manipulation.