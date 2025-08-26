+++
date = '2025-08-26T13:01:03-05:00'
draft = true
title = 'Know Your Imports'
+++
In the C# world, I've seen a tendency to just grab libraries off of nuget and use them without investigating them. Much like I see in the node.js universe, we seem to pull dependencies based on:

1. How many thumbs up it has
2. If we're feeling extra dilligent, checking how many stars the GitHub repo has.

While relying on community reviews is mostly probably fine, I've ran into cases where the community overlooks bugs and bad implementations. What follows is one such tale.

A couple years ago, I was asked to implement a very basic profanity filter based on a "bad words" list. I didn't argue how dumb of an idea this was, as it would have just ended with the business being frustrated, and me being angry, tired, and still implementing the feature. The main question is how to implement this while not punishing our users too heavily. The tool I turned to is bloom filters.

### A Brief Overview of Bloom Filters

I won't go into too much technical detail here. If you want that, there's a great [Wikipedia article](http://en.wikipedia.org/wiki/Bloom_filter) that goes into all the fun detail. The important point is that this is a great filter for seeing if a word is **not** in a list of bad words, and is relatively cheap compared to an actual list check. The great things about Bloom Filters are:

1. They are relatively easy to implement
2. They are performant
3. They are memory efficient compared to the size of the input list.

I've included a very simple implementation below, commented for clarity:

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

From the Wikipedia article, the false positive rate of this is determined by three factors:

1. The number of different hashing algorithms used.
2. The size of the storage array, and
3. The number of items hashed

### Finding a library

Not liking that we had "low level" code in our codebase, our boss went and imported a bloom filter library and refactored the code to use it. However, there was something odd about the re-implementation. I don't remember it perfectly, but the code looked something like:

```csharp
using Algorithms = BloomFilter.HashAlgorithms
class ProfanityFilter
{
	private BloomFilter f1;
	private BloomFilter f2;

	public ProfanityFilter(List<string> badwords){
	
		f1 = new BloomFilter(2000, 0.02, Algorithms.Murmur32BitsX86, badwords);
		f2 = new BloomFilter(2000, 0.02, Algorithms.Murmur32BitsX86, badwords);
	
	}

	public bool IsNotProfanity(string word){
		return !f1.Contains(word) || !f2.Contains(word);
	}
}
```

Now, there are a few problems with this code:

1. The same hashing algorithm is specified twice, effectively doubling the amount of work for no gain.
2. This doubles the amount of memory required by the filter, making it less effective.

The most interesting thing about this, however, is the fact that you only specify one hashing function. As for what the rest of the numbers mean:

1. 2000 - this is the number of expected elements.
2. 0.02 - this is the desired error rate.

Now, the library does something clever by using data the user most likely has a better grasp on, and uses those to set the number of hashing algorithms and array size. However, there are a few problems with this approach:

### The obvious problem

The first problem that comes to mind is that it isn't transparent to the user what impact the number of items and/or desired error rate have on memory consumption. This can be a problem when trying to use this in memory constrained environments, like AWS Lambda functions. It also makes it difficult for the programmer to reason through potential issues, like:

1. We said that we expect to have 2000 elements, but in reality, we have 10,000. How does this impact performance?
2. After increasing the limit to 10,000 elements, AWS is giving out of memory errors. Why?

Now, learning a bit about bloom filters, combined with reviewing the code for the library, will help answer these questions. However, I've been ignoring the sneakier problem.

### The less obvious problem

Remember how we only specify one function. Well what the program does is compute multiple hashes using that one function, and sets accordingly. The clearest example of the code is in:

```csharp
using System;

namespace BloomFilter.HashAlgorithms;

/// <summary>
/// Building a Better Bloom Filter" by Adam Kirsch and Michael Mitzenmacher,
/// https://www.eecs.harvard.edu/~michaelm/postscripts/tr-02-05.pdf
/// </summary>
public class Murmur32BitsX86 : HashFunction
{
    public override HashMethod Method => HashMethod.Murmur32BitsX86;

    public override long[] ComputeHash(ReadOnlySpan<byte> data, long m, int k)
    {
        long[] positions = new long[k];
        var hash1 = Internal.Murmur32BitsX86.HashToUInt32(data);
        var hash2 = Internal.Murmur32BitsX86.HashToUInt32(data, hash1);
        for (int i = 0; i < k; i++)
        {
            positions[i] = (hash1 + i * hash2) % m;
        }
        return positions;
    }
}
```

It is interesting that they cite an academic paper about bloom filters, so we can get some of the rationale behind their decisions. However, even in the paper, they require two separate hash functions in order to guarantee their conclusion that multiple hash functions can be simulated with just two.

### The problem with this approach

The main problem with this approach is that hashes are ultimately deterministic, not random. To go through a hypothetical example, let's use M as shorthand for Murmur32BitsX86.HashToUInt32, and assume:

M("bat") = 32 and M("cat") = 32.
Then, as hashes are deterministic:

M("bat", 32) is equal to M("cat", 32);

and going through the loop, the positions set would be:

	M("bat") + 0 * M("bat",M("bat"))
	M("bat") + 1 * M("bat",M("bat"))
	...
	M("bat") + (n-1) * M("bat",M("bat"))

Which means that the false error rate for any number of hashes is always the same as the false error rate for one hash.

### Who Cares?

Ultimately, you may not care about this kind of error. However, it underscores the point that when choosing a library, we shouldn't be blind to the implementation details. After all, this was low stakes. The next time may not be.
	
	
