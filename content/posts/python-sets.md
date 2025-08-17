---
date: '2025-08-16T14:40:33-04:00'
draft: true
title: 'Python Sets - A Reminder of their Power'
---

### Lightning-Fast Lookups!

Python sets are a useful alternative to lists, with their main advantage being that they only allow unique members. My most common use case is calling set() on an existing list to remove duplicates:

```
list1 = [1,4,22,1,5,5,2,394]

set1 = set(list1)
print(set1)

# (1,4,22,5,2,394)
```

Sets are useful enough for that alone, but I'd forgotten about another use case: instantaneous lookups, or membership checks. Typically if I'm checking if a value is in a group of items, I'll be working with that group as a Python list and check by with the `is in` syntax:

`is_present = 1 in list1`

For smaller datasets, this works well and fast enough. A recent project was different though - the group I was checking against had over two million items, and I had to check many values against that list. Running this looped code against a list took multiple minutes, and so I was spending Friday afternoon looking at ways to optimize.

To ensure I was only checking against unique items, I converted the list to a set. By using the same `in` syntax against the list, the whole loop ran in less than two seconds!  This confused me at first, because my logging showed that after converting to a loop there were no items removed - they were all unique values to begin with. Then why the speedup?

I remembered from my earlier days that a Python set is essentially a Python dictionary with no values assigned to the keys. Python dictionary lookup is very fast - when checking if a key is present in a dictionary, the code does not need to loop through all the other keys to see if that key exists. This is due to the underlying implementation of dictionaries (and sets) using the "hash table" data structure. When inserting a value to a dictionary or set, the code calls a hash function for the input value and stores the value at the location of the output hash. When looking up a dictionary value or checking for membership in a list, the same hash function is used and the value is returned from the location of output hash (or not, if the lookup value is not part of the group).

Whereas the list implementation needed to scan through two million items to see if there was a match, the set implementation just checks to see if there is anything stored at value's hashed location in the set. In bigO notation, this means that the set lookup is O(1), or constant time regardless the number of elements, while the list lookup is O(n) - scales with the n number of values in the list. 

I used Claude to generate a Python script to compare lookup times between lists and sets, and the difference for large datasets was even more drastic than I expected:

**Individual Lookups**

- 10,000 items  
  - List: 0.0047 ms avg  
  - Set: 0.000142 ms avg  
  - ✅ Set is ~313× faster  

- 1,000,000 items  
  - List: 10.215 ms avg  
  - Set: 0.000140 ms avg  
  - ✅ Set is ~72,965× faster  


**Looped Lookups**

- 10,000 lookups on dataset with 1,000,000 items  
  - List: 50.04 seconds total  
  - Set: 0.002 seconds total  

These numbers match my experience when switching from lists to sets for my actual use case. If you ever find yourself checking for membership in a large dataset, reach for a set - your future self (and any users of your code) will thank you.