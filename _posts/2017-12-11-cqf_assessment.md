---
layout: post
title: Counting Quotient Filter Assessment
published: true
---
Sketches provides approximate representation for a data using little amount of memory compared to the real data size. I am covering in this blog post two types: Approximate membership query(AMQ), and Counting Sketches. As the name suggests, AMQ answers if a particular element exists on a sketch or not. Counting sketch return how many times the element inserted in the sketch. I made assement for Quotient Sketches family [Quotient Filter](http://vldb.org/pvldb/vol5/p1627_michaelabender_vldb2012.pdf) and [Counting Quotient Filter](https://dl.acm.org/citation.cfm?id=3035963).I am comparing it with  [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter) and [Count min sketch](https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch).

The Idea of quotient filter was first coined in the first paper. The second paper  improved Quotient filter under the name of Rank Select Quotient filter(RSQF). It also developed a counting sketch(Counting Quotient Filter) based on the same ideas. The further analysis of the Quotient filter is based on the improved version of the quotient filter in the second paper. The experiments are done using [khmer package](https://github.com/dib-lab/khmer). Khmer implements the count min sketch and it implements a wrapper for the [cqf library](https://github.com/splatlab/cqf).

![QuotientFilter.jpg]({{ site.baseurl }}/images/QuotientFilter.jpg "qf")
Quotient Filter(QF) is approximate membership query data structure.Like Bloomfilter, QF doesn't produces true negative errors. In other words, If the item doesn't exist in QF, we are certain that the item doesn't exist in the original set. The Above figure discribes the insertion algorithm in the QF. First, the filter splits the hasbits into two components: quotient and the remaining part(fingerprint). Quotient Part is used to determine the target slot. Fingerprint is inserted into the target slot.

Insertion Algorithm produces two type of collisions. Soft Collision, When two items have the quotient but the remining part is different. Linear probing is used to find the next slot. Since linear probing doesn't work well in space tight conditions. Original QF add 3 metadata bits per slot to help linear probing to find the next slot.

RSQF uses less metadata bits(2.125).


Counting Quotient Filter(cqf) is uses the same insertion strategy as RSQF; however, It allows counting the number of instances inserted. If the item inserted for the first or second time, the remaining part is inserted in the target slot. If the item is inserted for the third time, the slot used for inserting the item in the second time is converted to counter. counters can be expanded to accomdate big counts. CQF impelments special encoding technique for the counters to differentiate between the counters and the remaining parts of other items.


Upsides:
1. Quotient Family has better data locaility than bloom filter. Items are saved in one place. Therefore, Quotient sketches are effieceint  when stored on main memory since it produce fewer cache misses than bloom filter. Quotient sketches also perform well when stored on SSD disk.
2. Quotient Sketches can be merged easily, like merging sorted lists. Although bloom filters can be merged easily by using OR operation, Only same sized filters can be merged. In case Quotient Sketches, We can merge sketches of different sizes.
3. Sketches resizing can be implemented using merging feature by simply merging the sketch with bigger one.
4. CQF uses variable size counters. So, It is suitable for counting data following ([zipifan distribution](https://en.wikipedia.org/wiki/Zipf%27s_law)) where most the items occures one or two times.

Summary of the Downsides:
1. When inserted kmer to fully loaded sketch the code just fail. I can’t even catch an exception.
2. Sketch size can only be of power of two. If you want to increase the sketch size you need to double it.
3. Sketch uses variable length counter for each kmer with max 2 bytes. If the kmer count exceeds 65535. The counter overflow and the value resets([See Basic Test](#basic-test))
4. Sketches need to have the same number of hashbits to be merged([See Merging Issue](#merging-issue)).  
5. Resizing is not implemented in the cqf library, and It can’t be implemented using the current cqf library.([See resizing issue](#resizing-issue))
6. CQF produces larger counting errors, but less often than countmin sketch([See accuracy test](#accuracy-test)).
7. I also tested loading the cqf with load factors more than 95%, and It passed the test. All the paper calculations are made where the cqf is loaded at 95%, so I was trying to make sure it can be fully loaded without failing([See Load Factor test](#load-factor-test)).

## Basic Test
[Code](https://github.com/shokrof/khmer/blob/DibMaster/tests/test_CQF.py)

I used some testcases from khmer to test the new QFCounttable. The test cases cover simple count, save, loading, and counting big values. All the test cases passed except counting the big values. CQF dynamically allocate bigger counters for high frequent items.However, the largest counter is 2 bytes; therefore, It can count up to 65535. If we try to count more than 65535, the counter overflows and counting stars from the beginning.


## Merging Issue
Mod operation is always used before adding the hash value to the sketch. Khmer uses calculate hash value mod sketch range. Since Quotient filter must use the same hash functions to be merged, Only sketches with the same range can be merged. Same range constraint is more relaxed of same sizes. Sketches with bigger sizes but smaller fingerprint can be merged as long as the haves the same number of hashbits. 

From [Storage.hh](https://github.com/shokrof/khmer/blob/DibMaster/include/oxli/storage.hh): Line 438

![qfadd.png]({{ site.baseurl }}/images/qfadd.png "qf")

I did [experiment](https://github.com/splatlab/cqf/blob/master/main.c) to test the above theory. I tried to merge two sketches of different sizes. As expected the code fails at the merging function.




## Resizing Issue



If we need to increase the number of slots for resizing, we need to increase the q without decreasing cf.range(using the same number of hashbits). Changing cf.range is similar to changing the hash function. So, We can increase the q and decrease r and maintain cf.range constant before and after the resizing.

I found that the r is always set to 8(size +8 ). I tried to change it to lesser values(2-7), but the code fails the basic test. Therefore. I concluded we can't implement resizing technique described in cqf paper using the current cqf implementation. Unless this bug is fixed.

From [Storage.hh](https://github.com/shokrof/khmer/blob/DibMaster/include/oxli/storage.hh): Line 418

![QFStorageInit.png]({{site.baseurl}}/images/QFStorageInit.png)

I tried to use cqf merge function to merge two equally size sketches into a bigger one, but it fails.

## Accuracy Test
[Script](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/runTests.sh)

I am trying to compare the accuracy of khmer implementation of both cqf and count min sketch. 
1. I created a dataset of kmers
2. Load the sketch using the dataset
3. Calculate the accuracy of kmers exist in the sketch.
4. Calculate the accuracy of kmers don’t exist in the sketch.


### Dataset
I created a simulated dataset for testing CQF implementation in Khmer. I developed [script](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/generateSeq.py) to generate kmers with left skewed frequency([zipifan distribution](https://en.wikipedia.org/wiki/Zipf%27s_law)). 
The script generates 3 files: 
1. dataset: kmers to be counted.
2. TruekmerCount:kmers count in format “kmer\tcount”.
3. Unseen Kmers: kmers that don't exist in the previous dataset.

I generated 1M kmer of length 20 and 10K unseen kmers.Here is the frequency distribution for the 1M kmers
![data1000000.goldHist.png]({{ site.baseurl }}/images/data1000000.goldHist.png)

### Experiment:
I did the experiment  using different sketch sizes. Sketch size is approximate number for the actual size allocated in the memory.


I used simple accuracy measure. I calculate the absolute difference between the true count for the kmer and the observed count ([Accuracy Test Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/testPerfomance.py)). I drew boxplot for all the values ([Plotting Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/plotPerformanceBoxPlot.py)).

![data1000000.Exist.png]({{ site.baseurl }}/images/data1000000.Exist.png)

![data1000000.NonExist.png]({{ site.baseurl }}/images/data1000000.NonExist.png)

As Expected, The cqf has less errors than count min sketch, but the cqf errors are more scattered. 

## Load Factor Test
[RSQF Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/testLoadFactor.py),[CQF Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/testLoadFactorCQF.py)

![filtersBitsPerelement.png]({{ site.baseurl }}/images/filtersBitsPerelement.png)



In Table2 from [cqf paper](https://www3.cs.stonybrook.edu/~ppandey/files/p775-pandey.pdf),  Bit per element is inversely proportional to the load factor of the RSQF. For load factor 95%, RSQF is more space efficient than bloom filter for all fpr values less than 1/64. Here, I am testing loading capacity of cqf . 
The first obstacle is to calculate the load factor for the cqf. After reviewing cqf code, I found that the number of slots that are actually allocated is more than the number passed as input. First, cqf code adds 10*sqrt(nslots) to the number of slots. Second, the result is rounded up to the next number divisible by 64. 

I created QFCounttable by passing 8192 as input. 9152 slots are created. I gradually increase the load factor by inserting more unique kmers. The program succeeded to insert 9050 unique kmers(98% load factor). The number of hash collisions was 24.Code





CQF testing,I am trying to test the loading of cqf with uniformly distributed kmers. I repeated the above experiment but each kmer is repeated M times. I ran the experiments  with different values of M and recorded the maximum number of kmers that can be inserted.
Values of M are 2**i-1 for i in 1:16

![loadingCQF.png]({{ site.baseurl }}/images/loadingCQF.png)
