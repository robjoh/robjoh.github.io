---
layout: post
title: "Accessing Azure Service Bus with Node.js"
date: 2014-08-16 13:00:41 -0700
comments: true
categories: azure node.js
---
One of the services we are building makes use of Azure Service Bus and we decided that we'd like to monitor the lengths of some of the subscriptions. I've been wanting to learn Node.js and so this sounded like a simple and fun project to start with. For the purpose of this post, I used the [azure portal](https://manage.windowsazure.com) to create a new service bus namespace *azure-service-bus-nodejs* with a single topic *my-topic* that has a single subscription *my-subscription*. You can find the code on GitHub at https://github.com/robjoh/azure-service-bus-nodejs, at the end of each section I'm including a link to the respective commit.

###Getting Started 
After [installing Node.js](http://nodejs.org/download/) I use `npm init` to begin a new project. This command asks a few questions (name, version...) and creates a package.json file that we'll be making use of shortly. 

![package.json](/images/azure-node-npm-init.png "npm init")

<small>[commit d675e487](https://github.com/robjoh/azure-service-bus-nodejs/commit/d675e48768f85a25c211614f8a04b616b30a7b26)</small>

###Connecting to Azure Service Bus
Now let's install the [Azure sdk for node](https://github.com/Azure/azure-sdk-for-node) with `npm install azure --save`. The --save option adds azure to the dependencies section of package.json. With the azure sdk is installed, I next create a server.js file which will contain the following program.

```javascript
"use strict"

var namespace = 'azure-service-bus-nodejs',
    accessKey = '[Access key for this namespace]',
    azure = require('azure');

var client = azure.createServiceBusService(namespace, accessKey);

client.listTopics(function(error, result, response) {
    if (error) {
        console.log(error);
        return;
    }

    console.log(JSON.stringify(result, null, 3));
});
```

In this program, we import the azure sdk using node's *require()* function. Next we create a service bus client for the *azure-service-bus-nodejs* namespace and execute the listTopics function. If successful, the *error* parameter will be null and the *result* parameter will contain the list of topics. Running the program with `node server.js` outputs the details of *my-topic*.

![List of topics](/images/azure-node-list-topics.png "node server.js")

<small>[commit 32e56247](https://github.com/robjoh/azure-service-bus-nodejs/commit/32e56247885073a500fe78ce2a8ee2ae6fc7b314)</small>

###Make it a web service
Now that we have the basic code for retrieving the list of topics, let's expose that functionality over http.

```javascript
"use strict"

var namespace = 'azure-service-bus-nodejs',
    accessKey = '[Access key for this namespace]',
    azure = require('azure'),
    http = require('http');

var client = azure.createServiceBusService(namespace, accessKey);

var server = http.createServer(function(httpReq, httpResp) {
    client.listTopics(function(error, result, response) {
        if (error) {
            httpResp.writeHead(500, {'Content-Type': 'application/json'});
            httpResp.write(JSON.stringify(error, null, 3));
            httpResp.end();
            return;
        }

        var topics = result.map(function(topic) {
            return {
                name: topic.TopicName,
                totalSubscriptions: topic.SubscriptionCount,
                totalSize: topic.SizeInBytes 
            };
        });

        httpResp.writeHead(200, {'Content-Type': 'application/json'});
        httpResp.write(JSON.stringify(topics, null, 3));
        httpResp.end();
    });
});

server.listen(8080);
```

In this code we pull in node's [http module](http://nodejs.org/api/http.html) and create a server that listens on port 8080. *createServer()* takes a function with arguments for the http request and response. When the server receives a request we make the listTopics call. If an error occurs we will return a 500 with the details. On a successful call, we'll use the *map* function to return an array of topic information. The http request/response would look like this. 

```http
GET http://localhost:8080/ HTTP/1.1
Host: localhost:8080

HTTP/1.1 200 OK
Content-Type: application/json
Date: Sun, 17 Aug 2014 19:15:51 GMT
Content-Length: 95

[
   {
      "name": "my-topic",
      "totalSubscriptions": "1",
      "totalSize": "0"
   }
]
```
<small>[commit ae2a1440](https://github.com/robjoh/azure-service-bus-nodejs/commit/ae2a14404ed51d0beb73a068685d512e6442b3ae)</small>

###What about subscriptions
Next we'll add support to get the list of subscriptions for a topic. We'll now accept the topic name as the url path. The code has also been broken up into a few functions with clearer repsonsibilities such as getting the data, writing the response and better organization.

```javascript
"use strict"

var namespace = 'azure-service-bus-nodejs',
    accessKey = '[Access key for this namespace]',
    azure = require('azure'),
    http = require('http');

var client = azure.createServiceBusService(namespace, accessKey);

var server = http.createServer(function(httpReq, httpResp) {
    if (httpReq.url === '/') {
        getTopics(writeResult);
    } else {
        getSubscriptions(httpReq.url.slice(1), writeResult);
    }

    function getTopics(callback) {
        client.listTopics(function(error, result, response) {
            if (error) {
                callback(error)
                return;
            }

            var topics = result.map(function(topic) {                
                return {
                    name: topic.TopicName,
                    totalSubscriptions: topic.SubscriptionCount,
                    totalSize: topic.SizeInBytes 
                };
            });

            callback(null, topics);
        });
    }
    
    function getSubscriptions(topic, callback) {
        client.listSubscriptions(topic, function(error, result, response) {
            if (error) {
                callback(error);
                return;
            }

            var subscriptions = result.map(function(subscription) {
                return {
                    name: subscription.SubscriptionName,
                    totalMessages: subscription.MessageCount
                };
            });
            
            callback(null, subscriptions);
        });
    }

    function writeResult(error, result) {
        if (error) {
            httpResp.writeHead(500, {'Content-Type': 'application/json'});
            httpResp.write(JSON.stringify(error, null, 3));
            httpResp.end();
            return;
        }

        httpResp.writeHead(200, {'Content-Type': 'application/json'});
        httpResp.write(JSON.stringify(result, null, 3));
        httpResp.end();
    }
});

server.listen(8080);
```

So we can now request the list of subscriptions for a specific topic.
```http
GET http://localhost:8080/my-topic HTTP/1.1
Host: localhost:8080

HTTP/1.1 200 OK
Content-Type: application/json
Date: Sun, 17 Aug 2014 21:06:30 GMT
Content-Length: 73

[
   {
      "name": "my-subscription",
      "totalMessages": "0"
   }
]
```
<small>[commit 6a412273](https://github.com/robjoh/azure-service-bus-nodejs/commit/6a41227305bc20e6ba836b889b3bb09eb91a14b9)</small>

###Move service bus to a module
Now let's move the service bus specific code to a service-bus.js module. This will help reduce the growing mix of request/response handling and application logic. 

```javascript
"use strict"

var azure = require('azure');

function mapTopic(topic) {
    return {
        name: topic.TopicName,
        totalSubscriptions: topic.SubscriptionCount,
        totalSize: topic.SizeInBytes 
    };
}

function mapSubscription(subscription) {
    return {
        name: subscription.SubscriptionName,
        totalMessages: subscription.MessageCount
    };
}

function createClient(namespace, accessKey) {
    var client = azure.createServiceBusService(namespace, accessKey);

    return {
        getTopics: function (callback) {
            client.listTopics(function(error, result, response) {
                if (error) {
                    callback(error)
                    return;
                }

                callback(null, result.map(mapTopic));
            });
        },

        getSubscriptions: function (topic, callback) {
            client.listSubscriptions(topic, function(error, result, response) {
                if (error) {
                    callback(error);
                    return;
                }

                callback(null, result.map(mapSubscription));
            });
        },

        getSubscription: function (topic, subscription, callback) {
            client.getSubscription(topic, subscription, function(error, result, response) {
                if (error) {
                    callback(error);
                    return;
                }

                callback(null, mapSubscription(result));
            });
        }
    };
}

module.exports.createClient = createClient;
```
The [module system](http://nodejs.org/api/modules.html) allows us to move the map functions out but they will remain private to this module. Only things exported via `module.exports` are available externally and so we export a single method to enable client creation. To reference our module we simply `require('./service-bus.js')`. Now our server.js has been reduced to the following.

```javascript
"use strict"

var namespace = 'azure-service-bus-nodejs',
    accessKey = '[Access key for this namespace]',
    serviceBus = require('./service-bus.js'),
    http = require('http');

var client = serviceBus.createClient(namespace, accessKey);

var server = http.createServer(function(httpReq, httpResp) {
    if (httpReq.url === '/') {
        client.getTopics(writeResult);
    } else {
        var parts = httpReq.url.split('/');
        if (parts.length === 2) {
            client.getSubscriptions(parts[1], writeResult);
        } else {
            client.getSubscription(parts[1], parts[2], writeResult);
        }
    }

    function writeResult(error, result) {
        if (error) {
            httpResp.writeHead(500, {'Content-Type': 'application/json'});
            httpResp.write(JSON.stringify(error, null, 3));
            httpResp.end();
            return;
        }

        httpResp.writeHead(200, {'Content-Type': 'application/json'});
        httpResp.write(JSON.stringify(result, null, 3));
        httpResp.end();
    }
});

server.listen(8080);
```
<small>[commit 1ebd6fbf](https://github.com/robjoh/azure-service-bus-nodejs/commit/1ebd6fbf376cac48740692c0836c8e5e10b3eecb)</small>

###Easier http with Express
As the application has grown, the http request/response manipulation has been getting more complex and fraught with other issues such as ignoring http verbs. Let's pull in [express](http://expressjs.com/) and clean this up. As earlier, we pull in express with `npm install express --save` and `require('express')`. As shown in the following code, you can set up handlers for verb/template combinations. For example, app.get('/:topic') matches an HTTP GET for /my-topic. If you look closely you'll notice that the topic parameter has been placed into httpReq.params.topic saving us from all of that parsing. Other nice things are the chainability of the response and it's automatic json conversion (no more JSON.stringify).

```javascript
"use strict"

var namespace = 'azure-service-bus-nodejs',
    accessKey = '[Access key for this namespace]',
    serviceBus = require('./service-bus.js'),
    http = require('http'),
    express = require('express');

var client = serviceBus.createClient(namespace, accessKey);
var app = express();

app.get('/', function(httpReq, httpResp) {
    client.getTopics(writeResult(httpResp));
});

app.get('/:topic', function(httpReq, httpResp) {
    client.getSubscriptions(httpReq.params.topic, writeResult(httpResp));
});

app.get('/:topic/:subscription', function(httpReq, httpResp) {
    client.getSubscription(httpReq.params.topic, httpReq.params.subscription, writeResult(httpResp));
});

http.createServer(app).listen(8080);

function writeResult(httpResp) {
    return function(error, result) {
        if (error) {
            httpResp.status(500).send(error);
            return;
        }

        httpResp.status(200).send(result);
    }
}
```
<small>[commit 9a01d5d0](https://github.com/robjoh/azure-service-bus-nodejs/commit/9a01d5d03951f34a0c06aa189ab786cbde7d0c98)</small>

###Wrapping up
This was a lot of fun but barely scratches the surface of Node.js, express and service bus. Next time I will deploy this to an [Azure website](http://azure.microsoft.com/en-us/services/websites/) and integrate it with [Application Insights](http://msdn.microsoft.com/en-us/library/dn481095.aspx).