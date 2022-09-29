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

**NOTE**: `keep the **Bearer Token** after it is generated`.

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
we need several steps to import Twitter data to Neo4j. Starts with store static value, create constraints, until parsing data so we can load them.

### 3.1. Store static value in Neo4j
The first thing to do is store static value in Neo4j, which is our **Bearer Token**.
```
CALL apoc.static.set("twitter.bearer", "<bearer_token>")
```
For checking stored-static value with prefix `twitter` in Neo4j
```
RETURN apoc.static.getAll("twitter")
```

### 3.2. Create constraints
From our data model, we know there are 3 node labels and their primary key from response in Postman. So, we can create constraints using that informations.

Create constraints with type `NODE KEY` for labels Tweet and User with their primary key as node property.

```
CREATE CONSTRAINT tweet_id IF NOT EXISTS FOR (n:Tweet) REQUIRE (n.conversation_id) IS NODE KEY;
CREATE CONSTRAINT user_id IF NOT EXISTS FOR (n:User) REQUIRE (n.user_id) IS NODE KEY;
```

### 3.3. Inspect Twitter data in Neo4j

```
WITH apoc.static.getAll("twitter") AS twitter
WITH "g20" as keyword, "202209280000" as fromDate, "202209290000" as toDate, twitter
WITH "https://api.twitter.com/1.1/tweets/search/30day/twitter.json?maxResults=100&query="+keyword+"&fromDate="+fromDate+"&toDate="+toDate as uri, twitter
CALL apoc.load.jsonParams(
  uri,
  {Authorization:"Bearer "+twitter.bearer},
  null
)
YIELD value
UNWIND value.results as status
return status limit 5
```
the query result is
<p align="center">
  <img src="https://user-images.githubusercontent.com/98151352/193114415-d321fa93-9211-4620-8f1f-acd5448cf454.png" />
</p>

### 3.4. Parse data and load data to Neo4j

```
WITH apoc.static.getAll("twitter") AS twitter
WITH "g20%20indonesia" as keyword, "202209280000" as fromDate, "202209290000" as toDate, twitter
WITH "https://api.twitter.com/1.1/tweets/search/30day/twitter.json?maxResults=100&query="+keyword+"&fromDate="+fromDate+"&toDate="+toDate as uri, twitter
CALL apoc.load.jsonParams(
  uri,
  {Authorization:"Bearer "+twitter.bearer},
  null
)
YIELD value
UNWIND value.results as status

MERGE (n:Tweet{conversation_id:status.id})
ON CREATE SET
n.tweet=status.text,
n.created_at=status.created_at,
n.retweet_count=status.retweet_count,
n.favorite_count=status.favorite_count

/// user post
MERGE (u:User{user_id:status.user.id })
ON CREATE SET
    u.name= status.user.name,
    u.screen_name=status.user.screen_name,
    u.description=status.user.description,
    u.followers_count=toInteger(status.user.followers_count),
    u.following_count=toInteger(status.user.friends_count),
    u.account_created_at=status.user.account_created_at,
    u.favourites_count=toInteger(status.user.favourites_count),
    u.verified=status.user.verified,
    u.profile_image_url=status.user.profile_image_url
MERGE (u)-[:POST]->(n)

// Hashtags Tweet
FOREACH (i in status.entities.hashtags |
    MERGE (hs:Hashtags{text:i.text})
    MERGE (n)-[:TAG]->(hs)
)

///user mention
FOREACH (
    i in status.entities.user_mentions |
    MERGE (um:User{user_id:i.id})
    ON CREATE SET 
        um.name=i.name,
        um.screen_name=i.screen_name
    MERGE (n)-[rm1:MENTIONS]->(um)
)


///  reply
FOREACH(
    ignoreMe IN CASE WHEN status.in_reply_to_status_id is not null 
    THEN [1] ELSE [] END | 
    MERGE (urp:User{user_id:status.in_reply_to_user_id})
    ON CREATE SET
        urp.screen_name=status.in_reply_to_screen_name
    MERGE (urp)-[:REPLY]->(n)
)

// Retweet
FOREACH(
    ignoreMe IN CASE WHEN status.retweeted_status.id is not null THEN [1] ELSE [] END | 
    MERGE (rt:Tweet{conversation_id:status.retweeted_status.id})
    ON CREATE SET
        rt.created_at=status.retweeted_status.created_at,
        rt.tweet=status.retweeted_status.text,
        rt.retweet_count=status.retweeted_status.retweet_count,
        rt.favorite_count=status.retweeted_status.favorite_count
    MERGE (n)-[:RETWEET]->(rt)
    
    // Hashtags Retweet
    FOREACH (i in status.retweeted_status.entities.hashtags |
        MERGE (hs1:Hashtags{text:i.text})
        MERGE (rt)-[:TAG]->(hs1)
    )

    // Retweet post
    FOREACH(
        ignoreMe IN CASE WHEN status.retweeted_status.user.id is not null THEN [1] ELSE [] END | 
        MERGE (u_rt_po:User{user_id:status.retweeted_status.user.id })
        ON CREATE SET
            u_rt_po.name= status.retweeted_status.user.name,
            u_rt_po.screen_name=status.retweeted_status.user.screen_name,
            u_rt_po.description=status.retweeted_status.user.description,
            u_rt_po.followers_count=toInteger(status.retweeted_status.user.followers_count),
            u_rt_po.following_count=toInteger(status.retweeted_status.user.friends_count),
            u_rt_po.account_created_at=status.retweeted_status.user.account_created_at,
            u_rt_po.favourites_count=toInteger(status.retweeted_status.user.favourites_count),
            u_rt_po.verified=status.retweeted_status.user.verified,
            u_rt_po.profile_image_url=status.retweeted_status.user.profile_image_url
        MERGE (u_rt_po)-[:POST]->(rt)
    )

    // retweet mention
    FOREACH (
        i in status.retweeted_status.user_mentions |
        MERGE (u_rt_mt:User{user_id:i.id})
        ON CREATE SET u_rt_mt.name=i.name, 
            u_rt_mt.screen_name=i.screen_name
        MERGE (rt)-[:MENTIONS]->(u_rt_mt)
    )

    // retweet repply
    FOREACH(
        ignoreMe IN CASE WHEN status.retweeted_status.in_reply_to_status_id is not null THEN [1] ELSE [] END | 
        MERGE (u_rt_rp:User{user_id:status.retweeted_status.in_reply_to_user_id})
        ON CREATE SET
            u_rt_rp.screen_name=status.retweeted_status.in_reply_to_screen_name
        MERGE (u_rt_rp)-[:REPLY]->(n)
    )
)
```

After the data is loaded, we can check the schema
```
CALL db.schema.visualization()
```
<p align="center">
  <img src="https://user-images.githubusercontent.com/98151352/193117051-76d32045-6975-407e-8ebe-120354146f65.png" />
</p>

and the data in Neo4j,
<p align="center">
  <img src="https://user-images.githubusercontent.com/98151352/193117610-6fc9279b-5a3e-4317-9e91-c96ae1a5de49.png" />
</p>

finally we can start to analyze the data ...

**NOTE** : `the data is ONLY loaded from first page of the Twitter API response`
