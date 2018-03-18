Not long ago, python introduced the concept of Single Dispatch function in python 3.4. It is basically a way to implement function
overloading using singledispatch decorator in functools and [pep443](https://www.python.org/dev/peps/pep-0443/) describes it in detail. It is really cool stuff but all the resources that I found elsewhere 
seemed to make it complicated, so I'll try to explain it in simple terms, it's usecases and how best to make the most out of it.

Let's first see a complicated nested conditional construct.
```python
def parse_time(time_string):
    if isinstance(time_string, pandas.Timestamp):
        return time_string.to_pydatetime()
    elif isinstance(time_string, pandas.Series) and 'datetime64' in str(time_string.dtype):
        return np.array([dt.to_pydatetime() for dt in time_string])
    elif isinstance(time_string, pandas.DatetimeIndex):
        return time_string._mpl_repr()
    elif isinstance(time_string, datetime) or time_format == 'datetime':
        return time_string
    elif isinstance(time_string, date):
        return datetime.combine(time_string, time())
    elif isinstance(time_string, tuple):
        return datetime(*time_string)
    elif time_format == 'utime' or isinstance(time_string, (int, float)):
        return datetime(1979, 1, 1) + timedelta(0, time_string)
    elif isinstance(time_string, np.datetime64):
        return _parse_dt64(time_string)
    elif isinstance(time_string, np.ndarray) and 'datetime64' in str(time_string.dtype):
        return np.array([_parse_dt64(dt) for dt in time_string])
    elif time_string is 'now':
        return datetime.utcnow()
    elif isinstance(time_string, astropy.time.Time):
        return time_string.datetime
    else:
      ...
```
This is a fucntion used by astronomers that takes in input a time string and parses it into a proper time object while being very flexible
with how the input is passed. As you can see, this greater flexibility comes at cost of code being not so neat. Suppose now that a new
time format is decided and has to be implemented. It will make this code even more cluttered.

Here comes single dispatch.
Now a simpler example to explain how to use it.

```python
from functools import singledispatch
 
 
@singledispatch
def info(x):
    raise NotImplementedError('Unsupported type')
 
 
@info.register(int)
def _(x):
    print("The argument is of type ", type(x))
    print("x*5 = ",x*5)
 
@info.register(str)
def _(x):
    print("The argument is of type ", type(x))
    print(x+" is a str ")

@info.register(list)
def _(x):
    print("The argument is of type ", type(x))
    print(len(x))

@info.register(dict)
def _(x):
    print("The argument is of type ", type(x))
    print(x.keys())
```

What this does is, defines a function info, applies the single dispatch decorator on it and then registers the function definations
for other argument types.
You may now guess the output for something like
```python
>>> info(10)
The argument is of type int
x*5 = 50
>>> info("hi")
The argument is of type str
hi is a str
>>> info([1,2,3])
The argument is of type list
3
>>> info({'a':10,'b':100,'c':1000})
The argument is of type dict
['a','c','b']
```
Note however that if type matches none of the specified registered definations, the default defination will be used.
You may check the list of resgitered definations by
```python
>>> info.registery.keys()
dict_keys( [<class 'int'>,<class 'str'>, <class 'list'>, <class 'dict'>, <class 'object'>])
```

Also note, one shortcoming of single dispatch is that, it dispacthes based on type of first argument. This means that conditionals with 
more than one variable can get complicated with single dispatch.
More powerful constructs can be created by using custom classes and usinng their name as type for dispatching.

```python
class Angle:
  value = 2*math.pi
  
  def __str__(self):
      return "I'm an angle with value '{}'".format(self.value)
  def todegree(self):
      return self.value/math.pi*180
      
class Distance:
  value = 10
  
  def __str__(self):
      return "I'm a distance with value '{}' km".format(self.value)

@singledispatch      
def info(object):
    print("I don't know this object :/ ")

@info.register(Angle)
def angleinfo(object):
    print("I'm '{}' degrees".format(object.todegrees()))

@info.register(Distance)
def distinfo(object):
  print(object)
```
Notice how we can use different names for the same function. This is actually a good practice to make the code more readable
and self explanatory.

End Notes
The main inspiration for Single Dispatch cmes from Lisp language which has really powerful controls for dispatching including multiple dispatch.
While single dispatcher is a great start, it would be nice to see this type of functionality for function overloading in python.






