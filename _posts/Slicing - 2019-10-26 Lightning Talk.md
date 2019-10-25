# Slicing
### TriPython Lightning Talk 2019-10-24

Recently, I was answering a question that involved the conversion of a string into a date. While coding a solution, I was admiring the ease of doing list splicing in Python. Looking for a syntax documentation, I noticed a __slice()__ item in the list of intrinsic functions. I was intrigued and wandered into the forest.

First, a brief description of the problem...
The data included string _dates_ in the form __yyyymmdd__. The user needed to do some date calculations, so that string needed to be converted into a date object.


```python
import datetime

sDate = '20191021'
dt = datetime.date(int(sDate[0:4]), 
                   int(sDate[4:6]),
                   int(sDate[6:8]))
print('explicit slices:\t', dt)
```

    explicit slices:	 2019-10-21
    

__Ta Daaaaa!__ Problem solved. Time to move on.

But what is this __splice()__ function? Can it simplify or enhance my splicing efforts? Time to explore this function.

In essence, the __splice()__ function returns an object that is described in a normal splice expression -- the colon-delimited numbers you see in the code above.


```python
slcYYYY = slice(0, 4)
slcMM = slice(4, 6)
slcDD = slice(6, 8)

dt = datetime.date(int(sDate[slcYYYY]), 
                   int(sDate[slcMM]), 
                   int(sDate[slcDD]))

print('slice vars:     \t', dt)
```

    slice vars:     	 2019-10-21
    

That works as advertised. What if we put slices into a list?


```python
slices = []
slices.append(slice(0, 4))
slices.append(slice(4, 6))
slices.append(slice(6, 8))

dt = datetime.date(int(sDate[slices[0]]), 
                   int(sDate[slices[1]]), 
                   int(sDate[slices[2]]))

print('slice list:     \t', dt)
```

    slice list:     	 2019-10-21
    

Ok. I hope you can see where I'm going with this - list comprehensions.


```python
print('list comp:      \t', [int(sDate[slc]) for slc in slices])
```

    list comp:      	 [2019, 10, 21]
    

But there's a problem :-(
The __datetime.date()__ method only accepts three separate parameters, not a list of three values.


```python
import sys

try:
    print(datetime.date([int(sDate[slc]) for slc in slices]))
except Exception:
    print('\nTrapped error feeding list comp to date method')
    print(sys.exc_info(), '\n')
    pass
```

    
    Trapped error feeding list comp to date method
    (<class 'TypeError'>, TypeError('an integer is required (got type list)'), <traceback object at 0x00000000051A6CC8>) 
    
    

This is where the `*` operator can help by unfolding the values in the list comprehension. Many thanks to Chris Calloway for expanding my understanding of this operator.


```python
dt = datetime.date(*[int(sDate[slc]) for slc in slices]) 
print('slice comp:      \t', dt)
```

    slice comp:      	 2019-10-21
    

In case you're wondering...yes, we can parse the date with the regular expression object.

Did you really think I'd pass up an opportunity to use regular expressions?!? (scoff)


```python
import re
dateParse = re.compile(r'^(\d{4})(\d\d)(\d\d)$')

dt = datetime.date(*[int(grp) for grp in dateParse.match(sDate).groups()])
print('regex parse:      \t', dt)
```

    regex parse:      	 2019-10-21
    

## Benefits?
* The individual explicit list slicing is the most common. If you used this method, your code would be understood by almost every Pythonista.

* I find the splice variables to be a bit more self documenting than the explicit slices.

* Just because we can put slices into an array doesn't mean that the result is any more readable, as we see in the individually indexed items example.

* However, we __can__ use Pickle to persist the list.

* I really like list comprehensions.

* Yaay regular expressions.

### What about performance?
Maybe there is a difference after all. Let's do a performance test.

As you can see below, the __slice vars__ method is a slight performance winner in addition to being self-documenting code.

Rounding out the top three are __slice list__ and __explicit slices__ methods.

Even though I like list comprehensions, this is 34% slower than __explicit slices__.

Regular expressions came in last, 1.35% slower than __explicit slices__. Better luck next time, buddy.


```python
import timeit

t = timeit.timeit("datetime.date(int(sDate[0:4]),int(sDate[4:6]),int(sDate[6:8]))", 
              setup="import datetime; sDate = '20191021'"
             )
print('explicit slices:\t', t)

#=========================================
t = timeit.timeit("datetime.date(int(sDate[slcYYYY]),int(sDate[slcMM]),int(sDate[slcDD]))", 
    setup="""
import datetime
sDate = '20191021'
slcYYYY = slice(0, 4)
slcMM = slice(4, 6)
slcDD = slice(6, 8)
"""
             )
print('slice vars:     \t', t)

#=========================================
t = timeit.timeit("datetime.date(int(sDate[slices[0]]),int(sDate[slices[1]]),int(sDate[slices[2]]))", 
    setup="""
import datetime
sDate = '20191021'
slices = []
slices.append(slice(0, 4))
slices.append(slice(4, 6))
slices.append(slice(6, 8))
"""
             )
print('slice list:     \t', t)

#=========================================
t = timeit.timeit("datetime.date(*[int(sDate[slc]) for slc in slices])", 
    setup="""
import datetime
sDate = '20191021'
slices = []
slices.append(slice(0, 4))
slices.append(slice(4, 6))
slices.append(slice(6, 8))
"""
             )
print('slice comp:     \t', t)

#=========================================
t = timeit.timeit("datetime.date(*[int(grp) for grp in dateParse.match(sDate).groups()])", 
    setup="""
import datetime
import re
sDate = '20191021'
dateParse = re.compile(r'^(\d{4})(\d\d)(\d\d)$')
"""
             )
print('regex parse:     \t', t)

```

    explicit slices:	 1.7844745119999885
    slice vars:     	 1.6614056800000014
    slice list:     	 1.7566811039999948
    slice comp:     	 2.461126089000004
    regex parse:     	 4.188342399999996
    
