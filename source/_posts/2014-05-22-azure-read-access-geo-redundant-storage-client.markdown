---
layout: post
title: "Azure read access geo-redundant storage client"
date: 2014-05-22 14:14:32 -0700
comments: true
categories: azure
---
On my current project we are working on our Business Continuity Plan for our various dependencies. One of those dependencies, Azure Blob Storage, already provides some redundancy with local or geo-redundant storage. However, you may still suffer some downtime while the failover process is being executed. Azure recently introduced the Read Access Geo-Redundant Storage feature which exposes a secondary endpoint which opens up access to one of the readonly replicas of your storage account. You can enable this on the configure page for your blob storage account and once setup the endpoint can be found on the dashboard. 

To use it from the blob storage client you just need to set the desired [LocationMode](http://msdn.microsoft.com/library/azure/microsoft.windowsazure.storage.retrypolicies.locationmode(v=azure.10).aspx). 

```c#
var account = CloudStorageAccount.Parse(connectionString);
var client = account.CreateCloudBlobClient();
client.LocationMode = LocationMode.PrimaryThenSecondary;
```

We will be using *PrimaryThenSecondary* for our query scenarios. So in the event of an outage the client automatically retries against the secondary replica and our app remains functional. Similarily, we'll be using this for robustness in asset delivery as our CDN's will be configured to use both primary and a secondary/fallback origin.
