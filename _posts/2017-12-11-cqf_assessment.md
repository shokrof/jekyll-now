---
layout: post
title: Counting Quotient Filter Vs Bloom filter and Countmin Sketch
---
I tested Counting Quotient Filter implemented in Khmer. The concept looks promising because of the new functionality offered by cqf: data locality, merging, and resizing. However, the implementation needs more work to be of similar quality of Khmer count min sketch. I will only include in this assessment about the downsides since the authors bragged about the upsides more than enough. 
Summary of the Downsides:
When inserted kmer to fully loaded sketch the code just fail. I can’t even catch an exception.
Sketch size can only be of power of two. If you want to increase the sketch size you need to double it.
Sketch uses variable length counter for each kmer with max 2 bytes. If the kmer count exceeds 65535. The counter overflow and the value resets(See Basic Test)
CQF produces larger counting errors, but less often than countmin sketch(See accuracy test).
Resizing is not implemented in the cqf library, and It can’t be implemented using the current cqf library.(See resizing issue)
I also tested loading the cqf with load factors more than 95%, and It passed the test. All the paper calculations are made where the cqf is loaded at 95%, so I was trying to make sure it can be fully loaded without failing(See Load Factor test).

