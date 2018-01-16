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

Insertion Algorithm uses a variant of linear probing to resolve collisions. If we are trying to insert item to occupied slots, linear probing uses the next vacant slot. Linear probing is very simple, and have good data locality, but It works well only when the load factor is low([reference](https://en.wikipedia.org/wiki/Linear_probing#Analysis)). In Space tight conditions, long contagious occupied slots (runs) decreases the performance of both inserting and query items. QF overcomes linear probing shortcoming by two changes. First, it keeps items in the run in sorted order. Second, it uses metadata to determine the start and the end of the runs. QF uses 3 metadata bits per slot.


## RSQF and CQF

[Prashant Pandey et al](https://dl.acm.org/citation.cfm?id=3035963) introduced significant improvements to the QF idea. They developed two new filters: Rank and Select Quotient Filter (RSQF) and Counting Quotient Filter (CQF). RSQF differs from QF into two major points. First, RSQF uses different metadata scheme than QF. Instead of 3 bits per slot, RSQF uses only 2.125 bits per slot. RSQF also deploys rank and select methods to speed up the search process. Unlike classical QF that slows down when the load factor exceed 70%, RSQF works efficiently up to high loading capacity (~95%). Second, RSQF splits the filter into blocks to store the metadata close to their slots. Therefore, RSQF has better data locality than QF.

CQF uses the same insertion strategy as RSQF, however it allows counting the number of instances inserted. If the item inserted more than once, enough slots immediately following that element’s remainder are used to encode for its count.

## Quotient Family Advantages
1. QF has better data locality than bloom filter. Items are saved in one place. Therefore, QF are efficient when stored in main memory since it produces fewer cache misses than bloom filter. QF also perform well when stored on SSD disk.
2. QF can be merged easily, like merging sorted lists. Bloom filters can be merged easily as well by using OR operation but only if they have the same size. In case of QF, we can merge filters of different sizes.
3. QF resizing is possible in streaming fashion.
4. RSQF uses less metadata thanQF, and it performs well when the filter is filled up to 95% of it's capacity.
5. CQF uses variable size counters. So, It is suitable for counting data following ([zipifan distribution](https://en.wikipedia.org/wiki/Zipf%27s_law)) where most the items occur one or two times.


## CQF Assessment
All the tests below can be found on my [khmer github repository](https://github.com/shokrof/khmer/tree/DibMaster/testsCQF).

### Install & Run
1. Clone test repo(git clone git@github.com:shokrof/khmer.git && git checkout DibMaster)
2. Install [Khmer](http://khmer.readthedocs.io/en/v2.1.2/dev/getting-started.html).
3. Install Parallel tool(sudo apt-get install parallel).
4. Install numpy and matplotlib(pip install numpy matplotlib).
5. cd testsCQF/ && run [runTests.sh](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/runTests.sh) script to generate the test data and run all tests.

### Tests:
1. [CQF Construction Test](#cqf-construction-test)
2. [CQF unit-test](#cqf-unit-test)
3. [Load factor Test](#load-factor-test)
4. [Accuracy Test](#accuracy-test)
5. [Merging & possible resizing test](#merging-and-possible-resizing-test)


### Dataset Description

Simulated datasets of 20 bp kmers were [designed](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/generateSeq.py) for testing CQF and comparing it with bloom filter and count-min sketch.
Simulated datasets include:
1. Zipifan dataset: 47M total kmers (1M unique kmers) following [zipifan distribution](https://en.wikipedia.org/wiki/Zipf%27s_law) (Figure 2)
2. Unique dataset: 1M unique kmers only.
3. TruekmerCount: kmers count in format “kmer\tcount”
4. Unseen dataset: 10K kmers that don't exist in the previous dataset.

![data1000000.goldHist.png]({{ site.baseurl }}/images/data1000000.goldHist.png)

Figure 2: the frequency distribution of the Zipifan dataset

### CQF Construction Test
CQF construction takes two values: number of slots(2^q), and number of hash bits(q+r). CQF [usage example](https://github.com/splatlab/cqf/blob/master/main.c) is used as template for testing. CQF example takes q value as input from user while r value is hard-coded to 8. CQF succeeds to construct filters using different values of q. However changing r value from 8 to any other value causes the code to fail.  

CQF current software can only construct filter whose r=8. After diving in the code, I found that the number of bits per slot is set at the compilation time. Consequently, CQF current software limits the remaining part size to number decided at the compilation time. This bug not only affects the flexibility of the software but also affects the merging and resizing feature, See [merging and resizing test](#merging-and-possible-resizing-test).

### Usage [Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/cqfConstruction.c):
~~~~
./cqfConstruction
~~~~


### CQF Unit Test
I borrowed some test cases from Khmer to test CQF. The test cases cover simple inserting/querying items into the filter, saving filter to hard disk, and loading from hard disk. All the test cases passed except inserting highly frequent items (>65535). CQF dynamically allocate bigger counters for high frequent items. However, the largest counter is 2 bytes; therefore, It can count up to 65535. If we try to count more than 65535, the counter overflows and restarts counting. According to the definition of the variable counter in CQF, it should be able to expand but at least it should maintain the maximum value and report to the user.


#### [Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/test_CQF.py) Usage:
~~~~
py.test tests/test_CQF.py
~~~~





### Load Factor Test

To find the maximum loading factor that can be achieved by CQF, a filter with known capacity was created then kmers from the unique dataset were inserted M times (Values of M are 2^i-1 for i in 1:16) until the filter fails. During insertions and to ensure accurate calculation of the load factor, kmers that might have collisions with the pre-inserted items were avoided. The load factor can be calculated by dividing the number of kmers inserted by the capacity of the filter. Our test CQF was created by passing 2^13 slots as an input. This filter should have 8192 slots but actually the current software created 9152 slots. Exploring the code showed that the constructor of the filter add 10*sqrt(Number of Slots). Authors [explain](https://github.com/splatlab/cqf/issues/4) that the extra slots are added to handle the overflow in the last block. The maximum numbers of unique kmers that can be inserted were recorded (figure 3 and figure 4).

![loadingCQF1-10.png]({{ site.baseurl }}/images/loadingCQF1-10.png)
Figure 3: Maximum Loading of CQF using uniform distribution.

![loadingCQF.png]({{ site.baseurl }}/images/loadingCQF.png)
Figure 4: Maximum Loading of CQF using uniform distribution(x-axis uses log scale).

CQF successfully inserted 9097 unique kmers(M=1) into the 9152 slots which leads to 99% load factor. With 1 fold increase in kmers repetition (M=2), The maximum number of unique kmers that can be inserted decreased to 49%. Another sharp reduction to 33% happened with another fold increase in kmers repetition (M=3). The decrease is due to CQF using slots to encode for counters. The reduction of the maximum allowed unique kmers continued gradually until the highest tested repetition (log2(M)=16). Two reasons might explain this continuous reduction:
1. CQF increases the counters’ size to use extra slots to handle the big counts which could explain the big reduction seen at log2(M)=8  
2. With gradual increase of M value, it is more likely to be bigger than the remaining value (r) encoding for the kmer. CQF has a special encoding scheme that add extra slot to tag these counters.  


#### [Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/testLoadFactorCQF.py) Usage
~~~~
python3 testloadFactorCQF.py <dataset of uniq kmers> <SketchSize(Must be power-of-2)>  <No Repeat>
~~~~



### Accuracy Test

I am trying to compare the accuracy of CQF, Bloom filter and count-min sketch wrapped in Khmer.

#### Experiment 1 Quotient Filter Vs Bloom filter
 Experiment 1 compares CQF and Bloom filter. Both filters are created with size approximate to 524k. Then, Unique kmers are iteratively inserted in both filters. Accuracy is measured periodically every 20,000 kmers. Accuracy measure is the number of false positives found while querying unseen kmers dataset.

 ![BloomVsCQF.png]({{ site.baseurl }}/images/BloomVsCQF.png)

Figure 5: Accuracy Comparison: Bloom filter Vs CQF

CQF's false positive grows slowly and steadily until the filter is completely filled. FPR values don't differ too much between empty and full filters. On the other hand, Bloom filter has lower FPR when the filter is near empty and half filled. After Saturation, FPR grows exponentially. Kmers can be further inserted into saturated bloom filter at the expense of higher FPRs.  


##### [Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/testSketchesAccuracy.py) Usage

~~~~
python testSketchesAccuracy.py <Unique Dataset> <Unseen Data Set>
~~~~


#### Experiment 2 (CQF Vs Count-min sketch):
Experiment 2 compares the accuracy of CQF and count-min sketch. Six instances of each structure were created to cover a range of different sizes (8M-268M). Same numbers of kmers from the Zipifan dataset were inserted into all data structures. Box plots are drawn for the differences between the true and observed counts of kmer queries using kmers from Zipifan (Figure 6) and unseen datasets (Figure 7).

![data1000000.Exist.png]({{ site.baseurl }}/images/data1000000.Exist.png)

Figure 6: Accuracy Comparison using the Zipifan dataset.

![data1000000.NonExist.png]({{ site.baseurl }}/images/data1000000.NonExist.png)

Figure 7: Accuracy Comparison using the unseen dataset.



The CQF has fewer but more scattered error compared to the count-min sketch.


##### Code Usage:
([Accuracy Test Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/testPerfomance.py)), ([Plotting Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/plotPerformanceBoxPlot.py))
~~~
parallel --gnu  -k "python3 testPerfomance.py <dataset_Prefix> {1} {2}" :::  <Sketch sizes(space seperated)>  :::  --cqf --cm > <dataset_Prefix>.result
python3 plotPerformanceBoxPlot.py <dataset_Prefix>
~~~


### Merging and possible resizing test
Quotient filters can be merged if they have been constructed using the same total number of hash bits. With the current implementation of CQF, merging two CQF into a new one was successful  as long as all of them have the same size.


[Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/mergeTest_SameSize.c) Usage:

~~~~
make
./mergeTest_SameSize
~~~~




Different QFs might use the same total number of hash bits but have different sizes because of using different quotient and remaining parts. These filters can be - theoretically - merged. Also resizing - which is not implemented in the current CQF library- can be done by merging the full small CQF into a larger empty one with the same no of hash bits but using smaller r value. As expected, testing of both ideas fail using the current implementation of CQF merge function because of the inability of the package to construct filters with different remaining parts


[Code](https://github.com/shokrof/khmer/blob/DibMaster/testsCQF/mergeTest_Resize.c) Usage:
~~~~
make
./mergeTest_Resize
./mergeTest_DifferentSize
~~~~


### Size Doubling
CQF uses power-of-2 sizes. It is very efficient method since bit shifting can be used to calculate modulus operation. Also, Dividing the hash values into quotient and remaining can be done using bit masks. However, it has two disadvantages.
1. Growing the CQF using size doubling technique is impractical for huge filter. For example, If you want to grow 32GB sketch, you will need 64GB which may not available.
2. It does not allow using prime sizes which is a usual best practice to avoids data clustering with unevenly distributed hash codes. [ref](http://srinvis.blogspot.com.eg/2006/07/hash-table-lengths-and-prime-numbers.html)


## Assessment Remarks
1. CQF has lower false positive rate than bloom filter when filters are highly loaded, see [Accuracy Test](#accuracy-test).
2. When the QF is fully loaded, the code just fail.
3. Sketch uses variable length counter for each kmer unti the count exceeds 65535. The counter overflow and the value resets ([CQF Unit Test](#cqf-unit-test))
4. Sketch size can only be of the power-of-two. If you want to increase the sketch size you need to double it ([See Size Doubling](#size-doubling)).
5. CQF need to have the same size to be merged([See Merging Issue](#merging-issue)).  
6. Resizing is not implemented in the CQF library, and It can’t be implemented using the current CQF library.([See resizing issue](#resizing-issue))
7. CQF produces larger counting errors, but less often than the count-min sketch ([See accuracy test](#accuracy-test)).
