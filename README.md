This repo is about social media analytics using Neo4j with Twitter data. 
It contains step-by-step from importing twitter data, parsing data, analyzing data to visualizing them.

## 1. Data Modeling

The data model is created in Neo4j Labs: [arrows.app](https://arrows.app/)
<p align="center">
  <img src="https://user-images.githubusercontent.com/98151352/193102289-dde868e3-3eb8-4f5c-8ee4-ced820ec952c.png" />
</p>

## 2. Twitter Data

We are using Twitter Premium v1.1 API. Therefore we need to have Twitter Developer Account and have premium access.

### 2.1. Authentication
Since we are using Twitter API, we need to provide Authentication for API methods. In this case, **Bearer Token** is being used for the authentication.

We can get our **Bearer Token** from Twitter Developer Portal.
<p align="center">
  <img src="https://user-images.githubusercontent.com/98151352/193104761-92672b84-b680-456a-8e95-d6b7b850d259.png" />
</p>

**note**: keep the **Bearer Token** after it is generated.

### 2.2. Premium v1.1: Search API

the url format is `https://api.twitter.com/1.1/tweets/search/30day/twitter.json?<parameters>` <br>
methods : `GET`

check available parameters [here](https://developer.twitter.com/en/docs/twitter-api/premium/search-api/api-reference/premium-search)

Let's see the response from API request first in **Postman**

url : `https://api.twitter.com/1.1/tweets/search/30day/twitter.json?maxResults=100&query=g20%20indonesia%20lang:en&fromDate=202209280000&toDate=202209290000`

Params :
- query : `g20%20indonesia%20lang:en`
- fromDate : `202209280000`
- toDate : `202209290000`
- maxResult : `100`

<p align="center">
  <img src="https://user-images.githubusercontent.com/98151352/193111160-dfdd174a-97f2-4eef-a66a-ed10c76c9aa6.png" />
</p>

in here our keyword is `g20 indonesia`

## 3. Import Data to Neo4j
we need several steps to import Twitter data to Neo4j. Starts with set up our params, create constraints, until parsing data so we can load them.

### 3.1. Set params in Neo4j
The first thing to do is set params in Neo4j, which is our **Bearer Token**.
```
CALL apoc.static.set("twitter.bearer", "<bearer_token>")
```

### 3.2. Create Constraints
From our data model, we know there are 3 node labels and their primary key from response in Postman. So, we can create constraints using that informations.

```
CREATE CONSTRAINT tweet_id IF NOT EXISTS FOR (n:Tweet) REQUIRE (n.conversation_id) IS NODE KEY;
CREATE CONSTRAINT user_id IF NOT EXISTS FOR (n:User) REQUIRE (n.user_id) IS NODE KEY;
```
