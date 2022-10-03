---
title: "Fast Document Similarity In Python with Minhash Locality Sensitive Hashing (MinHashLSH)"
layout: post
date: 2022-09-08 18:00
tag: python, lsh, minhash, similarity, forecasting
headerImage: true
usemathjax: true
projects: false
hidden: false # don't count this post in blog pagination
description: "How to implement fast document duplicate detection in python using locality sensitive hashing."
category: blog
author: nicods
externalLink: false
---

<!-- USED TO EXPORT MATH FORMULAS IN PDF FROM MARKDOWN

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
<script type="text/x-mathjax-config">
  MathJax.Hub.Config({ tex2jax: {inlineMath: [['$', '$']]}, messageStyle: "none" });
</script> -->


<!--<img class="image" src="{{ site.url }}/assets/images/keep-training/cover.png" alt="Cover Image"/>-->
In the big data era, it is always more frequent that companies need to detect similar items in their database. Imagine platforms like [Kijiji](https://www.kijiji.it/) or [Subito](https://www.subito.it/), trying to detect people that constantly duplicate announcements on the platform to boost their visibility without paying for sponsorship offered by the platform.
Assume for example the platform has 1000000 announcements and it wants to detect duplicates. If we try to check one by one all the pairs we need to check 499999500000 (half a trillion) pairs. If it takes a microsecond (0.000001 s) to check a pair, it would take 499999 seconds, that are approximately six days of computation. 
In this article, we are going to explore data mining techniques to speed up the whole process. The whole algorithm is inspired by [chapter 3 of the book "Mining of Massive Datasets"](http://www.mmds.org/#ver21).

# Dataset
In this guide we will use some announcements of the house for rent in Rome scraped from [Kijiji](https://www.kijiji.it/) a few years ago. You can download it from [here](https://github.com/nicoDs96/Document-Similarity-using-Python-and-PySpark/blob/master/LSH/dataset_rent_rome_kijiji.tsv).
The file is a tsv file with header:

|Title|Short Description|Location|Price (Euro)|Timestamp|Url Adv|
|--------------------|--------------------|--------------------|------------------|-------------------|--------------------|
|Studio accessoria...|Affitto studio a ...|Roma |450 | 12 ottobre, 11:32 |https://www.kijij...|
|Negozio 169Mq per...|Privato affitta n...|Prenestino / Casi...|1.700 | 12 ottobre, 08:45 |https://www.kijij...|
|Negozio in tiburt...|Negozio c/1 roma ...|Tiburtino / Colla...|6.000 | 17 October, 21:20 |https://www.kijij...|
|Studio medico via...|Studio medico avv...|Trieste / Somalia...|200 | 17 October, 20:22 |https://www.kijij...|

To read the [dataset](https://github.com/nicoDs96/Document-Similarity-using-Python-and-PySpark/blob/master/LSH/dataset_rent_rome_kijiji.tsv) we use the [pandas](https://pandas.pydata.org/) library:

```python
import pandas as pd
dataset=pd.read_csv("dataset_rent_rome_kijiji.tsv", sep="\t")
```
# In brief
In the following section we are going to see:
1. how to transform a document into a set using k-shingling
2. how to represent a set in a compressed way computing its signature in such a way that set similarity is preserved, using MinHashing.
3. how to implement a hashing function that hashes similar items in the same bucket, using LSH (locality-sensitive hashing).

# Shingling of Document
In literature, representing documents as sets is considered an effective way to identify lexically similar documents. The [k-shingles](https://en.wikipedia.org/wiki/W-shingling) method represents a document as a set of the substrings of length k. 
For example, if your document is 'I love pizza Margherita, a 6-shingle representation of the document based on characters, including spaces, can be `{'I love', ' love ', 'love p', 'ove pi', ...}`. According to the use case, you can compose shingles of words instead of characters or eliminate whitespaces and others.
In Python, we can implement a class that, given the parameter k and a document read into a string, returns the k-shingle set of a document:
```python
class shingler:
    def __init__(self, k):
        if k > 0:
            self.k = int(k)
        else:
            self.k = 10   
    #inner class utility
    def process_doc(self, document):
        return re.sub("( )+|(\n)+"," ",document).lower()
    
    def get_shingles(self, document):
        shingles = set()
        document= self.process_doc(document)
        for i in range(0, len(document)-self.k+1 ):
            shingles.add(document[i:i+self.k])
        return shingles
```
And use it as follows:
```python
>>> doc = "I love pizza Margherita!                     xd 1111@\n\n\n\n\n\n\n\n\n\n\n\n\n\n"
>>> shingler_instance = shingler(10)
>>> shingles_set = shingler_instance.get_shingles(doc)
>>> shingles_set
{' xd 1111@ ', 'herita! xd', 'izza margh', 'e pizza ma', 'rgherita! ', 'ta! xd 111', 'ita! xd 11', ' pizza mar', 'argherita!', 'rita! xd 1', 'zza marghe', 'a! xd 1111', 'a margheri', ' love pizz', 'erita! xd ', 'love pizza', 'za margher', 'margherita', 'gherita! x', ' margherit', 'i love piz', 'pizza marg', 've pizza m', 'ove pizza ', '! xd 1111@'}
```
Now we can check if two documents are similar using the Jaccard Similarity, a popular set similarity indicator:
$$ J(s1, s2) = \frac{|s1 \cap s2|}{|s1 \cup s2|} $$

In general, we might want to hash shingles to reduce their size, but even by hashing, they can take in a mean 4 times the size of the original document. So "what's the point?" you might think. Let me introduce you to MinHashing.

# Minhash - The signature of a set
The minash of a set can be seen as its signature, which is a unique and short representation of the set with a fixed length. The magic of MinHashing for a set is that it preserves Jaccard similarity (more or less).

We can represent a set with its characteristic matrix: a matrix whose columns are sets and rows are elements. The matrix contains a 1 in all the cells that correspond to an element contained in a set.
For example:  
S1 = {a, b}, S2 = {a, c} S3 = {a,b,c,d}  
  
| |S1|S2|S3|
|-|--|--|--|
|a| 1| 1| 1|
|b| 1| 0| 1|
|c| 0| 1| 1|
|d| 0| 0| 1|

Ideally, the minhash function takes a random permutation of the rows and for each set return the first element with a 1 in the characteristic matrix with permuted rows:
if the permutation is `badc`, the characteristic matrix is

| |S1|S2|S3|
|-|--|--|--|
|b| 1| 0| 1|
|a| 1| 1| 1|
|d| 0| 0| 1|
|c| 0| 1| 1|

and the minhash values are  
Minhash(S1) = b  
Minhash(S2) = a  
Minhash(S3) = b  

We obtain a signature of size n for the set if we compute minhash for n random permutations of the rows of the characteristic matrix. 

### Minhash in practice
The permutation approach is not feasible in practice, hence once again we use hash functions to approximate. We define a family of hash functions as
```python
import hashlib
# the family of hash functions, in this case, is the same function (sha1) applied with a different salt.
class hashFamily:
    def __init__(self, i):
        self.resultSize = 8 # how many bytes we want back
        self.maxLen = 20 # how long can our salt be (in decimal)
        self.salt = str(i).zfill(self.maxLen)[-self.maxLen:]
        
    def get_hash_value(self, el_to_hash):
        return int(hashlib.sha1(str(el_to_hash).encode('utf-8') + self.salt.encode('utf-8')).hexdigest()[-self.resultSize:], 16)
    
# NOTE: we use sha1 to avoid installing and importing an external library, sacrificing performances. No crypto-hash is required for this use case.
```
Then we compute a minhash signature for a set with the following algorithm:
1. Take the first hash function, and apply it to all of the shingle values in a document. Find the minimum hash value produced and use it as the first component of the MinHash signature.
2. Now take the second hash function, and again find the minimum resulting hash value, and use this as the second component. 
3. And so on...

So if we have 10 random hash functions, we’ll get a MinHash signature with 10 values for each set. We’ll use the same 10 hash functions for every document in the dataset and generate their signatures as well. 

```python
from random import randint, seed
class minhashSigner:
    def __init__(self, sig_size):
        self.sig_size=sig_size
        # Init the hash family of functions using a random salt
        self.hash_functions = [hashFamily(randint(0,10000000000)) for i in range(0,sig_size)]
    
    def compute_set_signature(self, set_):
        set_sig = []
        for h_funct in self.hash_functions:
            min_hash = math.inf
            for el in set_:
                h = h_funct.get_hash_value(el)
                if h < min_hash:
                    min_hash = h
                
            set_sig.append(min_hash)
        
        return set_sig
    #return a list of lists that can be seen as the signature matrix
    def compute_signature_matrix(self, set_list):
        signatures = []
        for s in set_list:
            signatures.append( self.compute_set_signature(s) )
        return signatures
```
Example usage:
```python
>>> doc = "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum."
>>> doc2 = "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident bla bla bla."
>>> shingler_inst = shingler(10)
>>> shinglings = shingler_inst.get_shingles(doc)
>>> shinglings2 = shingler_inst.get_shingles(doc2)
>>> minhash_sig = minhashSigner(20).compute_signature_matrix([shinglings,shinglings2])
>>> minhash_sig
[[1783860, 5303198, 4845898, 3935831, 198103, 5887007, 8061377, 4526224, 8467452, 7766250, 15505788, 825368, 12316643, 4502308, 6403903, 9259449, 8533874, 31889076, 940623, 278359], [1783860, 20203690, 4845898, 3935831, 198103, 12350289, 8061377, 4526224, 8467452, 7766250, 15505788, 825368, 12316643, 6122847, 6403903, 9259449, 8533874, 31889076, 940623, 278359]]
>>>
>>> set_sig_1 = set(minhash_sig[0])
>>> set_sig_2 = set(minhash_sig[1])
>>> jaccard_similarity_sig = len(set_sig_1.intersection(set_sig_2))/len(set_sig_1.union(set_sig_2))
>>> jaccard_similarity_sig
0.7391304347826086
>>>
>>> jaccard_similarity_shingle_set = len(set(shinglings).intersection(shinglings2))/len(set(shinglings).union(shinglings2))
>>> jaccard_similarity_shingle_set
0.8285077951002228
```
From the example, we can see that the Jaccard similarity between set signatures is near to the similarity between shingle sets. Of course, we can achieve a lower approximation error by tuning the shingle size and signature size.

# Locality Sensitive Hashing 
Quoting ["Mining of Massive Datasets"](http://www.mmds.org/#ver21)
> Locality-sensitive hashing (also known as near-neighbor search) is a general theory focused on how to approximatively find similar pairs without investigating all of them. The principle is that we are going to hash items several times in such a way that similar items are more likely to be hashed to the same bucket than dissimilar items are.
When you have a minhashed set a popular choice for LSH is the "banding technique", that is:
1. get the signature matrix and divide it into bands
2. hash each column of the band. The elements that hash to the same bucket are likely to be similar.
3. repeat for all the bands. You can use the same hash function of the previous band but you must use different buckets. 

<!--image here -->
In python:
```python
class lsh:
    def __init__(self, threshold=0.8):
        self.threshold = threshold
    def get_signature_matrix_bands(self, sig_matrix, bands_nr, sign_len): 
        #bands_nr = b
        #sign_len = n
        r = int(sign_len/bands_nr) #number of rows in each band
        bands = {} # {band_nr: [col_1,col_2,...]} where col_1 is all the values of Sig(S_i) for band b.
        for i in range(0,bands_nr):
            bands[i] = []
        # put Subsets of the columns of signature matrix into the appropriate bucket and cosider a column 
        # as a unique block so that we can hash the entire column.
        # Basically a band is a list of element, where each element is a subset of a signature of a given set.
        for signature in sig_matrix: 
            for i in range(0, bands_nr):
                idx = i*r    
                bands[i].append(' '.join(str(x) for x in signature[idx:idx+r]) )          
        return bands
    #band is a list 
    # construct a dictionary {hash(band_column): doc_id that produced this hash}
    def get_band_buckets(self, band, hash_funct):
        buckets = {}
        for doc_id in range(0,len(band)):
            value = hash_funct.get_hash_value( band[doc_id] )
            if value not in buckets:
                buckets[value] = [doc_id]
            else:
                buckets[value].append(doc_id)      
        return buckets
    def get_candidates_list(self, buckets):
        candidates = set()
        # buckets is a dictionary containing key=bucket, value= list of doc_ids that hashed to bucket
        for bucket,candidate_list in buckets.items():
            if len(candidate_list) > 1:
                for i in range(0,len(candidate_list)-1):
                    for j in range(i+1,len(candidate_list)):  
                        pair = tuple(sorted( (candidate_list[i],candidate_list[j]) ))
                        candidates.add(pair)
                
        return candidates #ie a set of couples, each couple is a candidate pair
    def check_candidates(self, candidates_list, threshold, sigs):
        similar_docs = set() #set of tuples
        # similar_pair is a couple containing doc_ids of documents that hashed to same bucket
        for  similar_pair in candidates_list:
            #for all the pairs of document in the list check similarity of their signatures
            doc_id_1 = similar_pair[0]
            doc_id_2 = similar_pair[1]
            signature_1 = set(sigs[doc_id_1]) #get the i-th column from signature matrix where i is doc_id in the collision list
            signature_2 = set(sigs[doc_id_2])
            js = len(signature_1.intersection(signature_2)) /len(signature_1.union(signature_2))
            if js >= threshold:
                similar_docs.add( tuple(sorted((doc_id_1,doc_id_2) )) )          
        return similar_docs
    def get_similar_items(self, sig_matrix, bands_nr, sign_len):
        similar_docs = set()
        #divide signature matrix into bands
        bands = lsh_instance.get_signature_matrix_bands(sig_matrix,bands_nr,sign_len)
        #for all the bands
        for band_id, elements in bands.items():
            #produce the buckets for the given band (band_id) with a random hash function
            buckets = lsh_instance.get_band_buckets(elements, hash_funct=hashFamily(randint(0,10000000000)))
            #Get all the candidate pairs
            candidates = lsh_instance.get_candidates_list(buckets)
            #Check all candidate pairs' signatures
            for sim_tuple in lsh_instance.check_candidates(candidates, self.threshold, sig_matrix):
                similar_docs.add( sim_tuple)
        return similar_docs #return all the similar signatures that respect the threshold
```
The class performs the following tasks:

1. Divide signature matrix into bands with `get_signature_matrix_bands(sig_matrix, bands_nr, sign_len)`. The method will return a list of strings where each string represents a column of the signature matrix for the given band. It is done to hash the entire column next when producing buckets.
2. Compute buckets for each band with `get_band_buckets(band, hash_funct)`, which will take as input a band `b` and a hash function `h` and will return a dict containing as key the bucket ids and as value a list of document ids that hashed to that bucket for the given band `b`: $\{bucket_{id}:[doc_i, doc_k, ...] \} $.
3. With `get_candidates_list(buckets)` we are going to take as input a list of buckets (whose union is the signature matrix) and will produce as output a set of tuples: those tuples represent documents that are hashed in the same bucket.
4. With `check_candidates(candidates_list, threshold, sigs)` we take all the candidates from all the bands and check if effectively their signatures produce a match with approximate Jaccard similarity greater than the threshold we give as a parameter.
5. With `get_similar_items(sig_matrix, bands_nr, sign_len)` we put together all the methods listed above and return the ids of similar documents that respect the threshold value.

The similarity threshold is passed as a value to the constructor of the class, default is 0.8.

Example usage:
```python
>>> shingler_inst = shingler(10)
>>> doc = "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum."
>>> doc2 = "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident bla bla bla."
>>>
>>> shinglings = shingler_inst.get_shingles(doc)
>>> shinglings = shingler_inst.get_hashed_shingles(shinglings)
>>>
>>> shingle2 = shingler_inst.get_shingles(doc2)
>>> shingle2 = shingler_inst.get_hashed_shingles(shingle2)
>>>
>>> signer = minhashSigner(100)
>>> signature_matrix = signer.compute_signature_matrix([shinglings,shingle2])
>>> lsh_instance = lsh(threshold=0.7)
>>> lsh_instance.get_similar_items(signature_matrix,10,100)
{(0, 1)}
```

# Benchmark Time
Now that we have implemented the whole code for fast approximate duplicate document detection, let's test it out against brute force detection and see what happens.
As the first step, we load the [dataset](https://github.com/nicoDs96/Document-Similarity-using-Python-and-PySpark/blob/master/LSH/dataset_rent_rome_kijiji.tsv) as a pandas data frame.
```python
print("Loading dataset...")
dataset=pd.read_csv("dataset_rent_rome_kijiji.tsv", sep="\t")
dataset['doc_id']=dataset.index
doc_nr = dataset['doc_id'].max()
print("Dataset loaded correctly.")
```
Then transform documents into k-shingles.
```python
print("Producing Shingles...")
start_time = time.time()
#an array where the index i represent the document_id and the element shingling_list[i] the hashed shingles for document document_id
shingling_list = [None] * (doc_nr +1) 
shingling_size = 10
signature_size = 50
bands_nr = 10

shingler_inst = shingler(shingling_size)

#produce hashed shinglings for all documents
for index, row in dataset.iterrows():
    doc = row['Title']+" "+row['Short Description']
    i = row['doc_id']
    
    shinglings = shingler_inst.get_hashed_shingles( shingler_inst.get_shingles(doc) )
    shingling_list[i] = shinglings

end_time = time.time()
print("Shingles produced in:\t %.2f seconds."%(end_time - start_time))
```
And we sign all the shingles and get the signature matrix.
```python
signer = minhashSigner(signature_size)
start_time = time.time()
print("Computing signature matrix...")
#produce a signature for each shingle set
signature_matrix = signer.compute_signature_matrix( shingling_list )
end_time = time.time()
print("Signature Matrix computed in:\t %.2f seconds." %(end_time - start_time))
```
In the end, we compute similar pairs with LSH.
```python
lsh_instance = lsh(threshold=0.8)
start_time = time.time()
print("Computing LSH similarity...")
lsh_similar_itemset = lsh_instance.get_similar_items(signature_matrix, bands_nr, signature_size)
end_time = time.time()
lsh_computation_time = end_time - start_time
print("LSH Similarity computed in:\t %.2f seconds.\nSimilar Elements Found: %d" %(lsh_computation_time,len(lsh_similar_itemset)))
```

We compute the similar pairs using brute force
```python
class bfsc():
    def compare_shingles_set_js(self, set1, set2):
        return len(set1.intersection(set2))/len(set1.union(set2))

brute_force_comparer = bfsc()
brute_force_similar_items = set()
start_time = time.time()

for i in range(0,doc_nr-1):
    for j in range(i+1,doc_nr):
        similarity = brute_force_comparer.compare_shingles_set_js(set(shingling_list[i]),set(shingling_list[j]))
        if similarity >= 0.8:
            brute_force_similar_items.add( (i,j) ) 
            
end_time = time.time()
bf_computation_time = end_time - start_time      
```

Check for similar pairs
1. LSH
```python
docs = lsh_similar_itemset.pop()
print("Doc 1:")
print(dataset.iloc[docs[0]] )
print("Is similar to\nDoc2:")
print(dataset.iloc[docs[1]] )
```
2. Brute Force
```python
docs = brute_force_similar_items.pop()
print("Doc 1:")
print(dataset.iloc[docs[0]] )
print("Is similar to\nDoc2:")
print(dataset.iloc[docs[1]] )
```
Print stats  
```python
report = ('LSH\n%.2f seconds to execute\n'
    '%d similar documents found\n\n'
    'Bruteforce search\n%.2f seconds to execute\n'
    '%d similar documents found.\n\n'
    'They find %d common similarities.')

print(report %(lsh_computation_time, len(lsh_similar_itemset),bf_computation_time,len(brute_force_similar_items),len(brute_force_similar_items.intersection(lsh_similar_itemset)) ))
```

# Conclusion
In this article we have seen:
1. how to transform a document into a set using k-shingling
2. how to represent a set in a compressed way computing its signature in such a way that set similarity is preserved, using MinHashing.
3. how to implement a hashing function that hashes similar items in the same bucket, using LSH (locality-sensitive hashing).
4. how to compare the approximate method with the brute force method.

All the theory behind the code here can be found in [chapter 3 of the book "Mining of Massive Datasets"](http://www.mmds.org/#ver21)

You can find the full example on [GitHub](https://github.com/nicoDs96/Document-Similarity-using-Python-and-PySpark/blob/master/LSH/DM_HW2_Ex2.ipynb). I wrote also a notebook with an Apache Spark implementation, available [on GitHub](https://github.com/nicoDs96/Document-Similarity-using-Python-and-PySpark/blob/master/LSH/DM_HW2_Ex2.ipynb).

Despite it is interesting to understand how that algorithm works, it is always a good idea to check for robust libraries to put this algorithm in production. A lot how implementations exist depending on the use case, here there are a fews:
* [Apache Spark Implementation](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.feature.MinHashLSH.html#pyspark.ml.feature.MinHashLSH)
* [datasketch](http://ekzhu.com/datasketch/)
* [LSH usage @ Uber](https://proxy-nyc.hidemyass-freeproxy.com/proxy/it-it/aHR0cHM6Ly93d3cudWJlci5jb20vYmxvZy9sc2gv)
* [Idealista implementation](https://github.com/idealista/tlsh-js)
* [Spotify implementation](https://github.com/spotify/annoy)
* [Facebook implementation](https://engineering.fb.com/2017/03/29/data-infrastructure/faiss-a-library-for-efficient-similarity-search/)
* [Benchmarks of some implementations](https://github.com/erikbern/ann-benchmarks)
