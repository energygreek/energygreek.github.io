---
draft: true
comments: true
date: 2025-05-13
tags: [random, c]
---


# generate random number
Recently, when analyzing a pcap of an RTP stream, I found that the SSRC of two streams was the same. After some research, I discovered that using time() as a seed for srand() can return the same result if called multiple times within a second. Using gettimeofday() provides higher resolution.

More importantly, srand() should be called only once in the main function!

## use time() as rand seed

```c
    srand(time(NULL));  // Set random seed using current time
    int r = rand();     // Get a pseudo-random number
    return 0;
```


## use gettimeofday() as rand seed

```c
void seed_rand_with_time() {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    // Combine seconds and microseconds for better entropy
    unsigned int seed = (unsigned int)(tv.tv_sec ^ tv.tv_usec);
    srand(seed);
}
```