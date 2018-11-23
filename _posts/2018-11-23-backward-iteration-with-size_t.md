---
layout: post
title: Backward iteration with unsigned index like size_t
date: 2018-11-23 00:30:00 +0100
tags: c++ size_t iterating
---

In C++11 the first tool for iterating a string or vector should be range-based
for-loop. But sometimes we really need the index of the element and we need 
the elements in reverse order. Index is usually needed when dealing with
`std::string`. Iterating with a signed integer is easy in both directions.
But if the vector size exceeds `INT_MAX`you have a logic error if you try to
iterate with signed index. Thus, **the most correct way is to use unsigned
integer**, `size_t`, as the type of the index.
Iterating forward with `size_t` is same as the signed variant, but iterating
backward can lead to infinite loop if you are not careful. The trick is to use
the not equal comparison.

```cpp
int main()
{
	auto v = vector<int>{1, 2, 3, 5, 7, 11};
	
	// backwards with greater than or equal is ERROR with unsigned
	//for(size_t i = v.size() - 1; i >= 0; --i)
	//	cout << v[i] << ' ';
	
	// backwards with not equal, OK, the right way
	for(size_t i = v.size() - 1; i != -1; --i)
		cout << v[i] << ' ';
		
	// trying to look smart. Correct, but may confuse.
	for(size_t i = v.size(); i --> 0;)
		cout << v[i] << ' ';
}
```

You may be wondering how can you compare unsigned number with a negative number
like -1. If you take into account the [wrapping nature of unsigned integers][1]
and the [implicit casting][2], then you will see that the comparison is correct.
It has the same effect as comparing `i != numeric_limits<size_t>::max()`.

[1]: https://en.cppreference.com/w/cpp/language/operator_arithmetic#Overflows
[2]: https://en.cppreference.com/w/cpp/language/operator_arithmetic#Conversions
