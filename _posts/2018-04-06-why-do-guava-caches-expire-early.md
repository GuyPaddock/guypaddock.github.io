---
layout: article
title: "Why do Guava Caches Expire Early? An Explanation"
categories: posts
modified: 2018-04-06T22:00:00-04:00
tags: [google, guava, caches, bugs, documentation]
comments: true
ads: false
---
Recently, I found myself scratching my head when using [caches](https://github.com/google/guava/wiki/CachesExplained) in Google's otherwise-excellent Guava library.

Consider this seemingly simple snippet of code:
```Java
  public static void main(String[] args) {
    Cache<Integer, String> cache = CacheBuilder.newBuilder().maximumSize(32).build();

    IntStream.rangeClosed(1, 33).forEach((value) -> {
      final List<Integer> valuesAfterAdd;

      System.out.println("Adding: " + value);
      cache.put(value, "Value " + value);

      valuesAfterAdd = new ArrayList<>(cache.asMap().keySet());

      Collections.sort(valuesAfterAdd);
      System.out.println("Values after add: " + valuesAfterAdd);
      System.out.println("Size after add: " + cache.size());
      System.out.println();
    });
  }
```

Now, without running the code, how many elements would you expect there to be in the cache at the end of the `forEach` loop? _31 elements? 32 elements? 33 elements?_

If you said 32 elements, you're right -- the loop iterates 33 times (since `rangeClosed` is inclusive of the upper bound), and the cache is set to limit the total number of elements to no more than 32. That all makes sense.

But, what is less intuitive is the answer to this question: _how many elements are there in the cache after you've added 32 elements?_ If you said 32, you'd be _wrong_.

Here's the output (shortened to highlight the juicy parts):
```
Adding: 1
Values after add: [1]
Size after add: 1

Adding: 2
Values after add: [1, 2]
Size after add: 2

Adding: 3
Values after add: [1, 2, 3]
Size after add: 3

... skipping ahead ...

Adding: 30
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30]
Size after add: 30

Adding: 31
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31]
Size after add: 31

Adding: 32
Values after add: [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32]
Size after add: 31

Adding: 33
Values after add: [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33]
Size after add: 32
```

30 elements.. 31 elements... then... 31 elements _again_? Then 32. _What the...?_ We're off by one, somehow. It's expiring the least-recently-used item too early.

To make things more confusing, this does _not_ happen if you choose a maximum size of 31:
```
... skipping ahead ...
Adding: 29
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29]
Size after add: 29

Adding: 30
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30]
Size after add: 30

Adding: 31
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31]
Size after add: 31

Adding: 32
Values after add: [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32]
Size after add: 31

Adding: 33
Values after add: [2, 3, 4, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33]
Size after add: 31
```

See? No longer off by one. Nor are we off by one with a maximum size of 33 (with a loop of 34 items):
```
... skipping ahead ...
Adding: 31
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31]
Size after add: 31

Adding: 32
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32]
Size after add: 32

Adding: 33
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33]
Size after add: 33

Adding: 34
Values after add: [1, 2, 3, 4, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34]
Size after add: 33
```

Nope, not off-by-one here...


Then, things get even stranger if you crank the maximum size of the cache up to 64 (and the loop to 65 elements):
```
... skipping ahead ...
Adding: 50
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
Size after add: 50

Adding: 51
Values after add: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51]
Size after add: 51

Adding: 52
Values after add: [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52]
Size after add: 51

Adding: 53
Values after add: [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53]
Size after add: 52

Adding: 54
Values after add: [3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54]
Size after add: 52

Adding: 55
Values after add: [3, 4, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55]
Size after add: 52

Adding: 56
Values after add: [3, 4, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56]
Size after add: 53

Adding: 57
Values after add: [3, 4, 6, 7, 8, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57]
Size after add: 53

Adding: 58
Values after add: [3, 4, 6, 7, 8, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58]
Size after add: 53

Adding: 59
Values after add: [3, 4, 6, 7, 8, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59]
Size after add: 54

Adding: 60
Values after add: [3, 4, 6, 7, 8, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60]
Size after add: 55

Adding: 61
Values after add: [3, 4, 6, 7, 8, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61]
Size after add: 56

Adding: 62
Values after add: [3, 4, 7, 8, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62]
Size after add: 56

Adding: 63
Values after add: [3, 4, 7, 8, 11, 12, 13, 14, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63]
Size after add: 56

Adding: 64
Values after add: [3, 4, 7, 8, 11, 12, 13, 14, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64]
Size after add: 56

Adding: 65
Values after add: [3, 4, 7, 8, 11, 12, 13, 14, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65]
Size after add: 57
```

*Now we're off by 8!*

As it turns out, it looks like [this is expected behavior](https://github.com/google/guava/issues/2442). Indeed, Google included this easy-to-miss blurb in the docs (presumably in response to #2442):
> If your cache should not grow beyond a certain size, just use CacheBuilder.maximumSize(long). The cache will try to evict entries that haven't been used recently or very often. _Warning:_ the cache may evict entries before this limit is exceeded -- typically when the cache size is approaching the limit.

_-- From [_Caches Explained_](https://github.com/google/guava/wiki/CachesExplained#size-based-eviction)_

If you step through the code, you'll find out that when using a maximum size of 32 with this particular cache configuration, the cache is split into two segments that each have a maximum length of 16 items. Depending upon which segment each new value falls into (by hash), the oldest item in that segment gets expired. It appears that a cache can have anywhere from 1 to 16 segments, and from what I can tell it seems like each segment is usually a power of 2 (from the code it appears that there's interplay with the concurrency level as well). So, you likely wouldn't see behavior unless your segments are close to multiples of 32.

The main reason for using segments is because it allows different threads to be working with the cache at the same time without blocking each other, if the values they're working with are in different segments. The cache need only to lock the segment in which the value falls, rather than locking the entire cache.

So, there you have it -- strangeness explained.
