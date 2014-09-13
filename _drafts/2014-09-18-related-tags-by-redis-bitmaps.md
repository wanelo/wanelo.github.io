---
layout: default
title: "Finding Related Tags using Redis"
---

Users on Wanelo can tag millions of products, and find interesting products through the tags. Suggesting related tags is useful to our users. 

One way to find related tags of a tag is to identify tags that also tag the same products. And collaborative filtering is doing just that. However, computing these related tags with quality and speed using collaborative filtering at scale is technically challenging. Below we introduce an innovative way to implement this algorithm using Redis and its bit operations. 

## Collaborative Filtering
Just a brief intro on [collaborative filtering](http://en.wikipedia.org/wiki/Collaborative_filtering). It is a common item-based recommender systems that recommend items based on users' rating or likes. A typical product recommendation would starts with a matrix with 2 entities and their relations between the two. Using figure 1 below as an example, the products are listed as rows and users as columns. The values are the ratings or likes for the user to the product. The matrix can be transposed without loss of generality, in which case the similar users would be recommended instead of products. 

For our use case (figure below), we have tags (items being recommneded) as rows and products as columns (features). And we treat 1 product taggging as 1 and no tagging as 0. 


![Product rating and taggings](/assets/ratings and taggings.png)

There are several ways to calculate the distance between 2 items. One common way is [Euclidean distance](http://en.wikipedia.org/wiki/Euclidian_distance). 

Another common method called [Jaccard index or Jaccard similarity coefficient](http://en.wikipedia.org/wiki/Jaccard_index) The Jaccard coefficient measures similarity between vectors, and is defined as the intersection divided by the union of the 2 vectors. 

![Jaccard index formula] (http://upload.wikimedia.org/math/1/8/6/186c7f4e83da32e889d606140fae25a0.png)

The second step is to compute a distance matrix between each items. Since our distance matrix is expressed 1s and 0s, 1 meaning a product is tagged by a tag, 0 meaning not. Jaccard index is a natural choice for us. To complete the distance, we need to compute their jaccard index for each pair of tags. 

![Similarity Score Matrix](/assets/similarity score matrix.png)

## Implementation

These are the basic tools and operation to build a collaborative filtering engine. The 4 steps are: 

  - Build redis bit vectors for each tag, with offsets being the product ids, and the value being 1 if tagged, or 0 if not.
  - For each pair of tags, compute the jaccard index by dividing the intersection by the union using redis bit operation, the result is the similarity score between 2 tags.
  - build and store a distance matrix (Jaccard index) for each pair of tags using sorted set.
  - Once the distance matrix is built, retrieving the top tags for a given tag is extremely easy and fast.

## Redis

Redis, differnt from other conventional in-memory storage, supports simple and advance data structures (String, List, Hashes, Sets and Sorted Sets) and commands to operate on them in-memory. One powerful data structure and operations it supports is bitmaps, an vector of 0s and 1s. 

### bitmaps

![Bitmap](/assets/bitmap.png)

And Redis allows users to set and get a bitmap by its offset. For example, if tag is called 'antique', and product id 7 is tagged with 'antique', then redis bit vector can be set like below: 

```
redis> GETBIT tag_antiuqe 7 
(integer) 0
redis> SETBIT tag_antiuqe 7 1
(integer) 0
redis> GETBIT tag_antiuqe 7
(integer) 1
```

Moreover, its provides powerful bit operation on these bitmaps. To compute the union and intersection between 2 bitmaps and store the result in the temporary key: 

```
redis> BITOP OR union tag_tall tag_brown
ok
redis> BITOP AND intersection tag_tall tag_brown
ok

redis> GET union
(integer) 20
redis> GET intersection
(integer) 10
```

### Sorted Set
Another powerful data structure is sorted set, which is similar to Redis Sets, with no repeating of string entries. Additionally, every member of a sorted set is associates with a score, which can be used to rank the members in the set. 

For example, once we created the Jaccard Index between 2 tags, we can store the score to a sorted set by:

```
redis> ZADD tag_brown_scores 0.7 'red'
(integer) 1
redis> ZADD tag_red_scores 0.7 'brown'
(integer) 1
```

Once we do this for all pairs of tags, we can retrieve the top related tags for 'brown' by:

```
redis> ZREVRANGE tag_brown_scores 0 -1
1) "red"
2) "yello"
3) "blue"
```

These are the basic tools and operation to build a collaborative filtering engine. The 4 steps are: 

  - Build redis bit vectors for each tag, with offsets being the product ids, and the value being 1 if tagged, or 0 if not.
  - For each pair of tags, compute the jaccard index by dividing the intersection by the union using redis bit operation, the result is the similarity score between 2 tags.
  - build and store a distance matrix (Jaccard index) for each pair of tags using sorted set.
  - Once the distance matrix is built, retrieving the top tags for a given tag is extremely easy and fast.

## Results

We selected about 3000 tags from half millions. And the results are pretty amazing. Some examples are: 


