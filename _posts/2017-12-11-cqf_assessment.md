---
layout: post
title: Counting Quotient Filter take over?
published: true
---
[Approximate membership query (AMQ)](http://www.cs.cmu.edu/~lblum/flac/Presentations/Szabo-Wexler_ApproximateSetMembership.pdf) data structures provide approximate representation for data using smaller amount of memory compared to the real data size. As the name suggests, AMQ answers if a particular element exists or not in a given dataset. Counting variants of AMQs return how many times the element was seen in the set.

Quotient filter (QF) is an AMQ that was first coined by [Michael A. Bender et al](http://vldb.org/pvldb/vol5/p1627_michaelabender_vldb2012.pdf) as an alternative to the commonly used Bloom filter to solve its chronic poor data locality. [Prashant Pandey et al](https://dl.acm.org/citation.cfm?id=3035963) published two enhanced versions of QF under the name of Rank Select Quotient filter(RSQF) and Counting Quotient Filter (CQF). [khmer](https://github.com/dib-lab/khmer) is a k-mer counting software that uses another AMQ called [count-min sketch](https://en.wikipedia.org/wiki/Count%E2%80%93min_sketch); a counting variant of [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter). Recently Khmer has implemented a wrapper for the [cqf library](https://github.com/splatlab/cqf) to test its performance. In this blog post I am using the Khmer wrapper to assess the CQF and compare it with Bloom filter and Count min sketch.


## Quotient Filter
![QuotientFilter.jpg]({{ site.baseurl }}/images/QuotientFilter.jpg "qf")

Figure 1: Quotient Filter

QF, like Bloom filter, doesn't produce false negative errors. In other words, If the item doesn't exist in QF, we are certain that the item doesn't exist in the original set. However, QF can produce false positive errors. For items having the same hash values, QF mistakenly report all of them exists if only one exists in the filter. The above figure describes the insertion algorithm in the QF. First, the filter splits the hash-bits into two components: quotient and remaining parts. Quotient Part is used to determine the target slot. The remaining is inserted into the target slot.

Insertion Algorithm uses a variant of linear probing to resolve collisions. If we are trying to insert item to occupied slots, linear probing uses the next vacant slot. Linear probing is very simple, and have good data locality, but It works well only when the load factor is low([reference](https://en.wikipedia.org/wiki/Linear_probing#Analysis)). In Space tight conditions, long contagious occupied slots(runs) decreases the performance of both inserting and query items. QF overcomes linear probing shortcoming by two changes. First, it keeps items in the run in sorted order. Second, it uses meta data to determine the start and the end of the runs. QF uses 3 metadata bits per slot while RSQF only uses 2.125 metadata bits per slot yet it is has better data locality.


## Counting Quotient Filter

CQF uses the same insertion strategy as RSQF; however, It allows counting the number of instances inserted. If the item inserted more than once, enough slots immediately following that element’s remainder are used to encode for its count.

## Counting Quotient Filter Advantages
1. CQF has better data locality than bloom filter. Items are saved in one place. Therefore, CQF are efficient when stored in main memory since it produces fewer cache misses than bloom filter. CQF also perform well when stored on SSD disk.
2. CQF can be merged easily, like merging sorted lists. Bloom filters can be merged easily as well by using OR operation but only if they have the same size. In case of CQF, we can merge filters of different sizes.
3. CQF resizing is possible in streaming fashion.
4. CQF uses variable size counters. So, It is suitable for counting data following ([zipifan distribution](https://en.wikipedia.org/wiki/Zipf%27s_law)) where most the items occur one or two times.


## CQF Assessment
All the tests below can be found on my [khmer github repository](https://github.com/shokrof/khmer/tree/DibMaster).

### Install & Run
1. Clone DibMaster branch
2. Install Khmer [guide](http://khmer.readthedocs.io/en/v2.1.2/dev/getting-started.html).
3. Install Parallel tool(sudo apt-get install parallel).
4. Install numpy and matplotlib(pip install numpy matplotlib).
5. cd testsCQF/ && run [runTests.sh](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/runTests.sh) script to generate the test data and run all tests.

### Tests:
1. [CQF unit-test](#cqf-unit-test)
2. [Load factor Test](#load-factor-test)
3. [Accuracy Test](#accuracy-test)
4. [Merging Test](#merging-test)
5. [Resizing Test](#resizing-test)


### Dataset Description

Simulated datasets were [designed](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/generateSeq.py) for testing CQF and comparing it with bloom filter and countmin sketch.
Simulated datasets include:
1. Zipifan dataset: 47M total kmers (1M unique kmers) of length 20 following [zipifan distribution](https://en.wikipedia.org/wiki/Zipf%27s_law) (Figure 2)
2. Unique dataset: 1M Unique kmers only.
3. TruekmerCount: kmers count in format “kmer\tcount”
4. Unseen dataset: 10K kmers that don't exist in the previous dataset.

![data1000000.goldHist.png]({{ site.baseurl }}/images/data1000000.goldHist.png)

Figure 2: the frequency distribution of the Zipifan dataset


### CQF Unit Test
I borrowed some test cases from Khmer to test CQF. The test cases cover simple inserting/querying items into the filter, saving filter to hard disk, loading from hard disk, and inserting items many times(> 65535 times). All the test cases passed except counting highly frequent items. CQF dynamically allocate bigger counters for high frequent items. However, the largest counter is 2 bytes; therefore, It can count up to 65535. If we try to count more than 65535, the counter overflows and counting stars from the beginning.

#### Code
Python unit test is used to implement the [test-cases](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/test_CQF.py)
 in khmer.

Usage:
~~~~
py.test tests/test_CQF.py
~~~~

#### Suggestion
1. qf_insert function should return false if the filter is fully loaded.
2. Counters should have maximum values to avoid overflow. 65535 is reasonable for kmer counting application.



### Load Factor Test

To find the maximum loading factor that can be achieved by CQF, a dataset with uniformly distributed kmer counts. CQF was created by passing 8192 as input. 9152 slots are created. In Every run, I iteratively insert kmers repeated M times and record the maximum number of unique kmers that can be inserted. Values of M are 2**i-1 for i in 1:16.Result is shown in the following graph.

![loadingCQF.png]({{ site.baseurl }}/images/loadingCQF.png)

Figure 3: Maximum Loading of CQF using uniform distribution.

#### Code
[CQF Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/testLoadFactorCQF.py)

Usage:
~~~~
python3 testloadFactorCQF.py <dataset of uniq kmers> <SketchSize(Must be power-of-2)>  <No Repeat>
~~~~



### Accuracy Test

I am trying to compare the accuracy of Khmer implementations of cqf,bloom filter, and count-min sketch.

#### Experiment 1 Quotient Filter Vs Bloom filter
 Experiment 1 compares Quotient Filter and Bloom filter. Khmer implementation of bloom filter is used, and CQF is used for the quotient filter because I couldn't find an implementation for RSQF. Bloom filter and CQF are created with size approximate to 524k. Then, Unique kmers are iteratively inserted in both filters. Accuracy is measured periodically. Accuracy measure is the number of false positives found while querying unseen kmers.

 ![BloomVsCQF.png]({{ site.baseurl }}/images/BloomVsCQF.png)

Figure 4: Accuracy Comparison: Bloom filter Vs CQF

 As Mentioned in the paper, Quotient filter is space efficient when the accuracy is below 1/64.

##### Code
[Bloom Test](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/testSketchesAccuracy.py)

Usage:
~~~~
python testSketchesAccuracy.py <Unique Dataset> <Unseen Data Set>
~~~~


#### Experiment 2 (CQF Vs Count-min sketch):
Experiment 2 compares the accuracy of cqf and countmin sketch. Kmers form Zipifan dataset is inserted into cqf and countmin sketch. Then Kmers from the same dataset and unseen dataset is queried. Box plot is drawn for the absolute difference of the true count and the observed count. The experiment is repeated using different sketch sizes. Results is shown in the box plot below.

![data1000000.Exist.png]({{ site.baseurl }}/images/data1000000.Exist.png)

Figure 5: Accuracy Comparison(same dataset used): Countmin vs CQF

![data1000000.NonExist.png]({{ site.baseurl }}/images/data1000000.NonExist.png)

Figure 6: Accuracy Comparison(unseen dataset used): Countmin vs CQF



As Expected, The cqf has fewer errors than count-min sketch, but the cqf errors are more scattered.


##### Code
([Accuracy Test Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/testPerfomance.py)), ([Plotting Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/plotPerformanceBoxPlot.py))

~~~
parallel --gnu  -k "python3 testPerfomance.py <dataset_Prefix> {1} {2}" :::  <Sketch sizes(space seperated)>  :::  --cqf --cm > <dataset_Prefix>.result
python3 plotPerformanceBoxPlot.py <dataset_Prefix>
~~~


### Merging Test
In Merge Test, I am going to dive deep in the merging capabilities of cqf. From Theoretical view, there is some constraints on the filters that must be met before merging. Mod operation is always used before adding the hash value to the sketch. Khmer uses calculate hash value mod sketch range. Since Quotient filter must use the same hash functions to be merged, Only filters with the same range can be merged. Same range constraint is more relaxed of same sizes. Filters with bigger sizes but smaller remaining parts can be merged as long as the haves the same number of hash-bits.

![qfadd.png]({{ site.baseurl }}/images/qfadd.png "qf")

Figure 7 [Storage.hh](https://github.com/shokrof/khmer/blob/DibMaster/include/oxli/storage.hh): Line 438

I did two experiments to test the above theory. First, I tried to merge two Quotient Filters with the same size into new one(also same size). CQF successfully merged the two filters. Then, I tried to merge two filters of different sizes. As expected the code fails at the merging function.

CQF can only merge filters using the same number of hashbits. Different size filters can be merged as long as they have the same number of bits; remaining parts is different in this case.

#### Code
Merging functions is not exposed to khmer library. So, I developed test cases using C linked directly to cqf library.
[Merge Test (Same Size)](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/mergeTest_SameSize.c),
[Merge Test (Different Size)](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/mergeTest_DifferentSize.c)


Usage:
~~~~
make
./mergeTest_SameSize
./mergeTest_DifferentSize

~~~~



### Resizing Test

I am testing the cqf resizing capabilities. Resizing is not implemented in the current cqf library. I am testing resizing using a workaround by merging full small cqf into empty big one.

Theoretically,If we need to increase the number of slots for resizing, we need to increase the q without decreasing cf.range(using the same number of hash-bits). Changing cf.range is similar to changing the hash function. So, We can increase the q and decrease remaining's size and maintain cf.range constant before and after the resizing.

I found that the r is always set to 8(size +8 ). I tried to change it to lesser values(2-7), but the code fails. I also tried to use cqf merge function to merge two equally size filters into a bigger one, but it fails.

Therefore. I concluded we can't implement resizing technique described in cqf paper using the current cqf implementation. Unless this bug is fixed.

#### Code
[Change R value test](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/mergeTest_Qbits.c) ,[Resize test](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/mergeTest_Resize.c).

Usage:
~~~~
make
./mergeTest_Qbits
./mergeTest_Resize
~~~~


#### Suggestion
CQF assign a byte array for the slots. It is better to uses variable size integer array to change the remaining size easily. SDSL library implements variable size integer arrays, Rank, and select data structures.
Also, Cache efficiency is over optimization in Kmer counting applications since it is I/O intensive application. CQF implementation will be simpler if the blocks organization idea is abandoned.


### Size Doubling
CQF uses power-of-2 sizes. It is very efficient method since bit shifting can be used to calculate modulus operation. Also, Dividing the hash values into quotient and remaining can be done using bit masks. However, it has two disadvantages.
1. Growing the CQF using size doubling technique is impractical for huge filter. For example, If you want to grow 32GB sketch, you will need 64GB which may not available.
2. Using Prime sizes avoids data clustering even when using simple bad hash functions. [ref](http://srinvis.blogspot.com.eg/2006/07/hash-table-lengths-and-prime-numbers.html)

## Assessment Remarks
1. Quotient filter is space efficient when the accuracy is below 1/64.
2. When inserted kmer to fully loaded sketch the code just fail. I can’t even catch an exception.
3. Sketch uses variable length counter for each kmer with max 2 bytes. If the kmer count exceeds 65535. The counter overflow and the value resets([CQF Unit Test](#cqf-unit-test))
4. Sketch size can only be of the power-of-two. If you want to increase the sketch size you need to double it ([See Size Doubling](#size-doubling)).
5. CQF need to have the same number of hash-bits to be merged([See Merging Issue](#merging-issue)).  
6. Resizing is not implemented in the cqf library, and It can’t be implemented using the current cqf library.([See resizing issue](#resizing-issue))
7. CQF produces larger counting errors, but less often than the count-min sketch([See accuracy test](#accuracy-test)).
