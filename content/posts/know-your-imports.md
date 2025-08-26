+++ 
draft = false
date = 2025-08-26T18:29:51-05:00
title = "Know your imports"
description = "A cautionary tale about knowing your imports"
slug = "know-your-imports"
authors = ["Sean McNabb"]
tags = ["C#","hashing"]
categories = []
externalLink = ""
series = []
+++
In the C#/.NET world, I've seen a tendency to just grab libraries off of NuGet and use them without much thought. Much like I see in the Node.js universe, we seem to pull dependencies based on:

1. How many downloads it has
2. Checking how many stars the GitHub repo has.
3. If we're feeling extra cautious, we may check how often it's updated, and when the last update was.

While relying on community reviews is mostly-probably fine, I've run into cases where the community overlooks bugs and bad implementations. What follows is one such tale.

### Profanity Filtering

A couple years ago, I was asked to implement a very basic profanity filter based on a "bad words" list. I didn't argue how dumb of an idea this was, as it would have just ended with the business being frustrated, and me being angry, tired, and still implementing the feature. The main question is how to implement this while not punishing our users too heavily. The tool I turned to is bloom filters[^1], as they are fast, memory efficient, and well suited for checking whether a word is NOT on a list.

The implementation of a simple bloom filter took about an hour, and worked well enough. However, our boss was not a fan of "re-inventing the wheel," and found a popular library that implemented bloom filters for us. 

### The "cleaner version"

It was a few years ago, and my memory is a little foggy, but the implementation code looked roughly like this:
```csharp
using Algorithms = BloomFilter.HashAlgorithms
class ProfanityFilter
{
	const int expectedEntries = 2000;
	const float desiredErrorRate = 0.02;
	private BloomFilter f1;
	private BloomFilter f2;

	public ProfanityFilter(List<string> badwords){
	
		f1 = new BloomFilter(expectedEntries, desiredErrorRate, Algorithms.Murmur32BitsX86, badwords);
		f2 = new BloomFilter(expectedEntries, desiredErrorRate, Algorithms.Murmur32BitsX86, badwords);
	
	}

	public bool IsNotProfanity(string word){
		return !f1.Contains(word) || !f2.Contains(word);
	}
}
```

This code looks reasonable, but there are a few issues.

1. The same hash function twice - double work for zero gain
2. Double memory - this creates two bitmaps instead of one.
3. Opaque parameters - it is difficult to reason about what impact a "desired error rate" has on the internals, which can lead to "fun" performance issues.

However, there is a more subtle problem, hinted at by how the filter is initialized. The library simulates multiple hash functions from just one.

### Why is that an issue?

Bloom filters rely on multiple independent hash functions. What the library did was take one hash function, mix it around a bit, and expect it to behave like it's several. The shortcut is based on a published academic article[^2], but the method described in that article requires two distinct hash functions.

The problem with this approach is that hashes are ultimately deterministic, not random. If two words collide on the first hash, and they are not employing double hashing[^3], they will collide on every derived hash too[^4].

In other words, it ends up doing a lot of extra work for minimal benefit [^5].

### So that's the whole story?

Happily, yes, in this case that's it. Ultimately, this was just a profanity filter, and the way the function was called was still performant, and good enough to satisfy the executives. But what if it wasn't such a low stakes scenario? For example, lets say we're using a bloom filter to check against a list of tens of thousands of compromised passwords. I can imagine a scenario where the added computation costs, being called often enough, can bring a server to a screeching halt.

### Takeaways

I think the primary takeaways from this fun excursion are:

- Don't blindly trust libraries - downloads, stars, and likes are not proofs of correctness.
- Understand what they are doing - no need to go over every detail, but at a high level, do you know how your libraries function?
- Understand why libraries are beneficial - libraries should help smooth over sharp implementation edges, not introduce confusing abstractions.

Ultimately, we are responsible for making sure that the software we ship works well for our end users. Can we really say we're owning that responsibility when we add library after library, trusting that there are no issues?

	


[^1]: I won't go into too much technical detail here. If you want that, there's a great [Wikipedia article](http://en.wikipedia.org/wiki/Bloom_filter) that goes into all the fun detail. The important point is that this is a great filter for seeing if a word is **not** in a list of bad words, and is relatively cheap compared to an actual list check. I've included a sample implementation below. It's a little naive, but the point is how simple it is to implement one.
    ```csharp
	public class BloomFilter(int size)  
	{  
	    private readonly BitArray _bitArray = new(size);  
	    private int _numberOfElementsHashed;  
	    private readonly List<IHashFunction> _hashFunctions =  
	    [  
	        MurmurHash3Factory.Instance.Create(),  
	        CityHashFactory.Instance.Create()  
	    ];   
	    public void AddToFilter(string item)  
	    {  
			// for each item, get a hash index, and set the bit to true
	        foreach (var hashFunction in _hashFunctions)  
	        {  
	            var idx =  GetBitIndex(item, hashFunction);  
	            _bitArray[idx] = true;  
	        }  
	        _numberOfElementsHashed++;  
	    }  
	  
	      
	    public bool NotInSet(string item)  
	    {  
			// assume not in set
	        var notInSet = false;  
	        foreach (var hashFunction in _hashFunctions)  
	        {  
				// get the computed index
	            var idx = GetBitIndex(item, hashFunction);  
				// if the bit is not set, we know the item is NOT
				// in the set
	            notInSet |= !_bitArray[idx];  
	        }  
	  
	        return notInSet;  
	    }  
	      
	    private int GetBitIndex(string item, IHashFunction algo)  
	    {  
	        var res = algo.ComputeHash(Encoding.UTF8.GetBytes(item));  
			// using UInt16 to avoid negative numbers when converting
			// back to int
	        var hashedInt = BitConverter.ToUInt16(res.Hash);  
			// ensure we keep the array in-bounds
	        return hashedInt % _bitArray.Length;  
	    }
	}
    ```

[^2]: https://www.eecs.harvard.edu/~michaelm/postscripts/tr-02-05.pdf

[^3]: Double hashing is a technique used in some hashing algorithms where the result of the first hash is the seed of the second hash. These are efficient, but are more highly correlated than two independent hashes from different algorithms.

[^4]:  To go through a hypothetical example, let's use M for the single hash function, and assume they collide:

        L = length of storage array.
        M("bat") = 32
        M("cat") = 32

	Furthermore, let's assume that in order to generate other hash functions, we derive them using the formula

		h1 = M(x)
		h2 = M(h1)
	
	Or more concretely, we can calculate as

	
	| i   | formula     | value    |
	| --- | ----------- | -------- |
	| 0   | h1 + 0 * h2 | 32       |
	| 1   | h1 + 1 * h2 | 32 + h2  |
	| 2   | h1 + 2 * h2 | 32 + 2h2 |
	| ... | ...         | ...      |
	
	Since these values are the same, these values % L are the same, and thus map into the same place in the bit array. This may seem odd, but I've seen something very similar to this implementation in at least one bloom filter package. For an example, see
    
    https://github.com/vla/BloomFilter.NetCore/blob/main/src/BloomFilter/HashAlgorithms/XXHash128.cs

[^5]: even if they are double hashing, the false positive rate may be higher than a bloom filter error rate formula would indicate. This renders the desired false positive error rate input for the function misleading.
