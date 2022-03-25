# reduce(…) in Python 3

Reduce built in was removed and moved to `functools` library. Change logs of Python 3 state: “Removed `reduce()`. Use `functools.reduce()` if you really need it; however, 99 percent of the time an explicit for loop is more readable.” [1].

Reduce takes three arguments – a reduce function that must accept two arguments, an iterable and an initial value. Quoting the official documentation [2]:

`functools.reduce(function, iterable[, initializer])`

> Apply function of two arguments cumulatively to the items of sequence, from left to right, so as to reduce the sequence to a single value. For example, reduce(lambda x, y: x+y, [1, 2, 3, 4, 5]) calculates ((((1+2)+3)+4)+5). The left argument, x, is the accumulated value and the right argument, y, is the update value from the sequence. If the optional initializer is present, it is placed before the items of the sequence in the calculation, and serves as a default when the sequence is empty. If initializer is not given and sequence contains only one item, the first item is returned.

Example of reduce that counts number of letters in a string:

```python
>>> string = 'hello world, this is a test string to test reduce in python3, even if it was moved to functools'
>>> def reduce_function(accumulator, item):
...     accumulator[item] = accumulator.get(item, 0) + 1
...     return accumulator
...
>>> from functools import reduce
>>> reduce(reduce_function, string, {})
{'h': 3, 'e': 8, 'l': 4, 'o': 8, ' ': 18, 'w': 2, 'r': 3, 'd': 3, ',': 2, 't': 11, 'i': 6, 's': 7, 'a': 2, 'n': 5, 'g': 1, 'u': 2, 'c': 2, 'p': 1, 'y': 1, '3': 1, 'v': 2, 'f': 2, 'm': 1}
```

It starts with an initial value of empty dictionary `{}`. Then for each element, the `reduce_function` is called with first argument being the dictionary and second being a letter. The function has to add the letter to the accumulating dictionary and return that dictionary, so it could be used as the first argument when the function is called for the second letter.

Adding print statement to print accumulator and item in `reduce_function` yields such output:

```python
{} h
{'h': 1} e
{'h': 1, 'e': 1} l
{'h': 1, 'e': 1, 'l': 1} l
{'h': 1, 'e': 1, 'l': 2} o
{'h': 1, 'e': 1, 'l': 2, 'o': 1}
```

[1] https://docs.python.org/3.0/whatsnew/3.0.html

[2] https://docs.python.org/3.0/library/functools.html#functools.reduce
