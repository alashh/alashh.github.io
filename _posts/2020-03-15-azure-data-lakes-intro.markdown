---
layout: post
title:  "Azure Data Lakes P1: My Plans"
date:   2020-03-06 17:43:32 -0500
categories: blog azure
tags: blog azure blob machine learning data lake hadoop hdinsight
---
I'm still learning a lot about data lakes, flows and ingestion. Almost every project I am involved in has some kind of data source, and those data sources are getting larger, and esoteric in their source (API, Stream, Relational!). It seems that we as an Industry have been able to collate and store hundreds of thousands of lines of data, but we've only now in the past few years been able to query it in an efficient way.

Some of the teams i work with build and train models with Machine Learning. They have 10-15 different data sources, ranging from Kafka Streams, Relational Databases and APIs. They use those data sources to build models with custom algorithms. While we as a team get involved with certain bits and pieces of the flow I feel great value from getting a view of the whole process. To this end i want to build my own end-to-end solution, and once built create an application to query said data.

I want to build this in a cloud provider as well, so to run this one on Azure. I've spent the past few posts working on AWS, so lets switch it up and run this one on Azure.


## The Tools
So, from my starter angle from Azure, the tools it seems we'll be using from them are:

| **Tool**                     | **Usage**                     |
|------------------------------|-------------------------------|
| Azure Data Lake Storage Gen2 | Storing Data                  |
| Azure Data Factory           | Orchestration of Data Ingress |
| HDInsight                    | Hadoop Cluster for Analysis   |

For now, i'm just going to be talking about data ingestion and querying. We'll get to the usage of said data in another multi stage section I decide to write.


## What Data?
So I need some data to start with from multiple different sources with multiple different ways to querying it, to get a good idea of how it's going to end up over in the land of the data lake. The whole thing needs a bit of *Raison d'être* as well, a reason to ingest and then analyse. For this ive decided to grab data from the past year on the following topics:

- Weather
- Formula 1 Results
- Historical Market Data for Title Sponsors to the Formula 1 Teams in Europe.

From this, I think there's enough differential all over the show, but what are we going to do with this data?


## What Goals?
Now that I have some data, what do i plan to do with the above. Where do I plan to go after i've got the data.

- Ingest all the relevant datasets into the blob storage
- Structure datasets for consumption
- Query datasets to build a cohesive relation between them all.

And the relation I want to find? 

***Effects of the weather on Formula 1 results and if those results correlated to stock prices of the Title Sponsor.***

Pretty strange kind of relation but hey; it's an excuse to give this all a shot with some easy to query data that I don't need to make myself. I also think as I go I might add or find more info or may change the whole thing to some cool idea I find along the way.

## What to do first?
1. I'll need to get the ingestion infrastructure up and running, this will be a collection of Azure Data Factory and the Blob Storage
2. Start learning how to query all these APIs for the data from Data Factory
3. Codify all the above into ARM or Terraform (gotta make this reproducible)
4. Find out how to make this not cost a fortune.

In the next post, i'll start building the Infrastructure, and start getting some of the data inside. So join me on stumbling through this stuff and listening to me ramble.