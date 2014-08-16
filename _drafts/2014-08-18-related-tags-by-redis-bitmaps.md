---
layout: default
title: "Compute Tag Cloud by collaborative filtering using Redis Bitmaps"
---

An user can tag arbitrary keywords onto millions of products on Wanelo, and also find interesting products through the tags. We thought it would be useful to also list out other related tags or keywords on the tag pages too. Computing these related tags with quality and speed is our technical challenge and the topic of this post. 

## collaborative filtering
Just a little background on [collaborative filtering](http://en.wikipedia.org/wiki/Collaborative_filtering). It is one of the most common type of item-based recommender systems that recommend related items based on users' rating or likes. A typical product recommendation would starts with a matrix with 2 entities and their relations between the two. Using figure 1 below as an example, the products are listed as rows and users as columns. The values are the ratings or likes for the user to the product. The matrix can be transposed withour loss of generality, in which case the similar users would be recommended instead of products. 

For our use case (figure below), we have tags (items being recommneded) as rows and products as columns (features). In more complicated use cases, we would have scalar values or have other columns as features. But in our simple use case, we would just treat 1 product taggging as 1 and no tagging as 0. 


![Product rating and taggings](/assets/ratings and taggings.png)

The second step is to compute a distance matrix or similarity score matrix between each items. There are many definitions or ways to calculate the distance between 2 items. 

One common way is [Euclidean distance](http://en.wikipedia.org/wiki/Euclidian_distance). In Cartesian coordinates, if p = (p1, p2,..., pn) and q = (q1, q2,..., qn), the distance between p and q is defined below. The lower the score, the more similar the two items are. 

![Euclidean distance formula] (http://upload.wikimedia.org/math/a/d/5/ad5e3fd5a68762ccdab67e5e0b71cb75.png)


Another common way is [Jaccard index or Jaccard similarity coefficient](http://en.wikipedia.org/wiki/Jaccard_index) The Jaccard coefficient measures similarity between finite sample sets, and is defined as the size of the intersection divided by the size of the union of the sample sets. The  shall be between 0 and 1; 0 being no similarity while 1 being complete similarity. 

![Jaccard index formula] (http://upload.wikimedia.org/math/1/8/6/186c7f4e83da32e889d606140fae25a0.png)

Because our data is expressed in binary, it is a natural choice for us. Now for each pair os items we just need to compute their jaccard index and fill the matrix. 

![Similarity Score Matrix](/assets/similarity score matrix.png)

---

## Redis

Enough math, let's talk a little about our tool Redis. Redis is a fast in-memory cache or storage. Different from othe conventional cache storage, it supports simple and advance data structures (String, List, Hashes, Sets and Sorted Sets) and commands to operate on them in-memory One powerful data structure and operations it supports is bitmaps, an vector of 0s and 1s. 

![Bitmap](/assets/bitmap.png)

And Redis allows users to set and get a bitmap by its offset. 

```
redis> GETBIT tag_key1 7 
(integer) 0
redis> SETBIT tag_key1 7 1
(integer) 0
redis> GETBIT tag_key1 7
(integer) 1
```

Moreover, its provides powerful bit operation on these bitmaps. To compute the union and intersection between 2 bitmaps and store the result in the designated key: 

```
redis> BITOP OR union tag_key1 tag_key2
ok
redis> BITOP AND intersection tag_key1 tag_key2
ok

redis> GET union
(integer) 20
redis> GET intersection
(integer) 10
```

Another powerful data structure is sorted set, which 




Now, we have the basic building tool for computing Jaccard Index between two items, which is essentiall the intersection divided by the union.


## Implmentaton

Based on our design, the tag to product matrix can be expressed in these bitmaps. One tag is created as one bitmap, with its offset being the product id, and offset value being 1 if tagged and 0 if not. This gives the raw matrix to work with. 

Then we loop through each pair of tags and compute the union and intersection of the pair, then we divide the two to get the Jaccard Index. Then we save the 

      


Results

We selected about 3000 tags from half millions. And the results are pretty amazing. Some examples are: 

