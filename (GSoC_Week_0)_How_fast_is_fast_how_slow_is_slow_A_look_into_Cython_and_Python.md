The scikit-learn `tree` module relies heavily on 
Cython to perform fast operations on NumPy arrays, so I've been learning the language (if you can even call it that) in order to effectively contribute.

At first, I was a bit skeptical about the purported benefits of Cython -- it's widely said that Python is "slow", but how slow is "slow"? Similarily, C code is known to be "fast", but its hard to get a grasp on the performance difference between Cython and Python without directly comparing them. This post summarizes a quick (and extremely unscientific) experiment I did comparing the performance of raw Python, Python code running in Cython, and Python code with static typing running in Cython. These results may not generalize to whatever application you have in mind for Cython, but they're suitable for seeing the existence of performance differences on a CPU-heavy task.

## Why would I want to use Cython?
Cython combines Python's ease of use with C performance to help developers optimize their Python code or create a fast Python interface to their C code.

To understand how Cython improves the performance of Python code, it is useful to have some knowledge of how code in Python and C is run. Python is a dynamically typed -- this means that variables do not have to be fixed at compile time, and a variable that starts as an `int` can be set to a `list` or even a custom Python object at any time. On the other hand, C is statically typed -- variable types must be defined at compile time, and they are generally that type and only that type. Also, Python is an interpreted language; this indicates that there is no compile step necessary to run the code. C is a compiled language, and files thus must be compiled before they are runnable.

Given Python's nature as a dynamically typed, interpreted language, the interpreter must spend time to figure out what type each variable is at runtime, extract the data from these variables, run the low-level machine instructions, and then place the result into a (possibly new) Python object that is returned. In C, the compiler can figure out at compile time all the details of low-level functions / data to use; a compiled C program spends almost all its runtime calling fast low-level functions, making it much faster than Python. Cython attempts to improve the performance of Python programs by bringing the static typing of C to Python, a dynamic language.

With a few exceptions, valid Python code is also valid Cython. To demonstrate what sort of speed gains are possible with Cython, we turn to the classic example of calculating fibonacci numbers.

### Python vs Cython
Below is a simple recursive function to calculate the nth Fibonnaci number in Python


```language-python
def fibonacci_py(n):
    a, b = 0, 1
    for _ in range(1, n):
        a, b = b, a + b
    return b
```

Let's see how long the Python function takes to calcuate several values of `fibonacci`


```language-python
%timeit fibonacci_py(0)
```
    1000000 loops, best of 3: 436 ns per loop



```language-python
%timeit fibonacci_py(70)
```


    100000 loops, best of 3: 4.89 µs per loop


Now, let's turn the above function into a Cython function without changing anything (remember that most valid Python code is valid Cython) and evaluate performance again.


```language-python
%load_ext Cython
```


```language-python
%%cython
def fibonacci_cy_naive(n):
    a, b = 0, 1
    for _ in range(1, n):
        a, b = b, a + b
    return b
```


```language-python
%timeit fibonacci_cy_naive(0)
```


    1000000 loops, best of 3: 227 ns per loop



```language-python
%timeit fibonacci_cy_naive(70)
```


    100000 loops, best of 3: 2.1 µs per loop


Now let's add static typing to the naive Cython code.


```language-python
%%cython
def fibonacci_cy_static(n):
    cdef int _
    cdef int a=0, b=1
    for _ in range(1, n):
        a, b = b, a + b
    return b
```


```language-python
%timeit fibonacci_cy_static(0)
```


    10000000 loops, best of 3: 59.3 ns per loop



```python
%timeit fibonacci_cy_static(70)
```


    10000000 loops, best of 3: 126 ns per loop


As you can see, it took Python 436 ns per loop to calculate `fibonacci(0)` and 4.89 µs per loop to calculate `fibonacci(70)`. Simply using Cython without any changes to the Python code more than doubled the performance, with 227 ns per loop to calculate `fibonacci(0)` and 2.1 µs per loop to calculate `fibonacci(70)`. However, the most dramatic performance increase came from using statically typed C variables (defined with `cdef`). Using statically typed variables resulted in 59.3 ns per loop when calculating `fibonacci(0)` and 126 ns per loop when calculating `fibonacci(70)`! In the case of calculating `fibonacci(0)`, this represents a 3x speed improvement over the naive Cython function and a 7x performance increase over the Python function. The speedup is even more pronounced when calculating `fibonacci(70)`; using statically typed variables gave a speedup of almost 17x from the naive Cython version and approximately a 39x improvement over the normal Python version!

Cython gives massive performance achievements on this simple fibonacci example, but it's worth nothing that this example is completely CPU bound. The performance between Python and Cython on a memory bound program would likely still be noticeable, but definitely not as dramatic as this toy example.

## Conclusion
While learning Cython, I wrote a [short iPython notebook tutorial](https://github.com/nelson-liu/cython-crash-course/blob/master/pointers.ipynb) on Cython pointers and how they work geared toward developers relatively fluent in Python but unfamiliar in C -- it's mainly intended to be practice / quick reference material, but you might find it handy if you want to learn more.

Additionally, the contents of the majority of this post are in an iPython notebook [here](https://github.com/nelson-liu/cython-crash-course/blob/master/why_use_cython%3F.ipynb).

For next week, I'll be providing a brief introduction to regression trees and some basic splitting criterion such as mean squared error (MSE) and mean absolute error (MAE).

If you have any questions, comments, or suggestions, you're welcome to leave a comment below :)

Thanks to my mentors [Raghav RV](https://github.com/raghavrv) and [Jacob Schreiber](https://github.com/jmschrei) for their constant support, and to the larger scikit-learn community for being a great place to contribute.

_You're awesome for reading this! Feel free to follow me on [GitHub](https://github.com/nelson-liu)  if you want to track the progress of my Summer of Code project, or [subscribe to blog updates via email](http://eepurl.com/b8NcW5)._
