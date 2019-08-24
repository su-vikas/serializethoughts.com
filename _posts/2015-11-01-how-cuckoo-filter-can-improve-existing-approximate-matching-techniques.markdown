---
layout: post
title:  "How Cuckoo Filters Can Improve Existing Approximate Matching Techniques"
date:   2015-11-01 16:17:22 +0800
---

If you have used [VirusTotal](https://virustotal.com), in additional information tab you will find a field for **ssdeep**. It is intriguing what this field represents among hashing functions SHA1, SHA256 and MD5. **ssdeep** is a approximate matching algorithm (AMA). NIST define [approximate matching algorithms](http://www.nsrl.nist.gov/Documents/ApproxMatchSP3-20130802.pdf) as:

```"Approximate matching is a technique for identifying similarities between or among digital objects, such as files, storage media, or network streams, which do not match exactly".```

In simple words, given two files, having certain degree of similarity, for example, two word files with second file being copy of first except a paragraph. Traditional hash functions can only tell about the exact similarity, but cannot provide degree or a confidence score out of 100 of how similar the files are.

AMA are used extensively in forensics investigation as they provide easy way to compare a hard disk against whitelisted or blacklisted corpus of files. Over past few years many AMA have been proposed. **ssdeep** was the first, followed by **sdhash** and **mrsh-v2**. In between, some other algorithms were also proposed, but they suffered with one shortcoming or other. All of the above algorithms use [Bloom filters](https://en.wikipedia.org/wiki/Bloom_filter) as their basic building block. In short, Bloom filter is a probabilistic data structure for performing set membership query efficiently. In this post I will not go into details of internal working of  AMA nor of Bloom filter, but talk about an alternative of Bloom filter, namely [Cuckoo filter](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf). Cuckoo filter is modification of Cuckoo hashing approach to construct a data structure suitable for set membership determination. Cuckoo filter excel over Bloom filter on following parameters:


- Cuckoo filter has better lookup performance
- Cuckoo filter has better space efficiency than Bloom filter, when false positive rate desired is < 3%.
- Unlike Bloom filter, Cuckoo filter supports deleting stored items dynamically.

In one of my recent works, I have used Cuckoo filter instead of Bloom filter in **mrsh-v2** and was able to achieve significant performance gain. The final results can be summarized as:

- In simple runtime performance, there is a gain of 37%.
- Size of final fingerprints generated is halved.
- Memory usage is 8th of approach with Bloom filter.

This work was presented at [ICDF2C-2015](http://d-forensics.org/), which took place on 6th October in Seoul, South Korea, titled [How Cuckoo Filter Can Improve Existing Approximate Matching Techniques](http://link.springer.com/chapter/10.1007/978-3-319-25512-5_4#page-1) ([pdf](/assets/docs/icdf2c.pdf)). The work was adjudged with best paper award by the committee.

In next coming posts I will try to cover what are approximate matching algorithms in detail and probably also look into each of these algorithms.

Keep hacking!!
