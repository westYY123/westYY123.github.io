---
layout: post
title: Cache constructed moka crate
date: 2022-09-07
categories: blog
tags: [Rust]
description: 
---

During my intern in HUAWEICLOUD, we have constructed a two layer cache using Redis as the second layer and a cache provided by moka crate.<br>
When Redis crashed, the try to access Redis will cause a panic, we just map_err of this try. However, we'd like to make this two layer cache can work without Redis which means when Redis is not available, this two layer cache can be a one layer cache.<br>
Hence, one of my work is to extract the error returned by Redis, this is a simple and explict task which can be completed and implemented easily.<br>
However, there is a fatal feature we have ignored when we construct this cache. Plz have a look at the code below:<br>
![pic](img/moka.png)
In the 4th line, we construct a cache with max capacity of 1, then the next two lines we insert two entries to this cache, as we know, the max capacity of our cache is 1, So the first one we insert is not available, but the if in 8th line indicate that we get "hyy1" this String success.<br>
Let's move on, in the 13th line, we call entry_count method to get the size of our cache, it returns us 0 which is confused.<br>
How can it be 0 after two insertion?<br>
If the max_capacity we specified can not limit the mount of entries, this can lead to OOM error. Imagining that endless insertion without elimination, the memory usage of this cache can be unlimited big.<br>
After reading of the doc and resource code, I find a method that called sync, this method will synchronize the cache. Then the entry_count return 1.<br>
![pic](img/moka2.png)
In other words, the cache constructed with moka already is a two layer cache, but I have no idea why this is designed to. All is in Memery, it doesn't make any sense.<br>
To avoid this potential error, I need to figure out is this can be unlimited big, so I inserted 1 million to my cache, and the memory usage is limited in a number. It doesn't cause OOM error.<br>
Then I wonder the capacity of this cache of cache. After test, the capacity is 800.