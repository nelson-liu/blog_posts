I've been a fan of the [`tqdm`](https://github.com/noamraph/tqdm) Python module for quite some time, but I found it difficult to find a reason to use it; generally, loops run fast enough that a progress bar is unnecessary. However, I found a perfect use for it in reading large files.

If the task isn't something I can speed up via `multiprocessing`, I can use `tqdm` to decide whether I can grab a cup of coffee or work on something else while I let it run. `tqdm` allows me to easily add a progress bar to the read operation, like so:
```python
f = open(file_path,'r')
for line in tqdm(f, total=get_line_number(glove_path)):
    # various operations here
```

As you can see, adding this functionality is as simple as wrapping the file with the `tqdm` method. However, to display the progress bar, `tqdm` needs an idea of how many total lines it needs to process. I use [this code snippet](http://stackoverflow.com/a/850962/1877942) from StackOverflow to quickly find this information when instantiating the progress bar:
```python
def get_line_number(file_path):
    fp = open(file_path, "r+")
    buf = mmap.mmap(fp.fileno(), 0)
    lines = 0
    while buf.readline():
        lines += 1
    return lines
```

Here's what it looks like in action:
<video  width="702" height="514" autoplay loop poster="http://thumbs.gfycat.com/ElasticSameAmericanrobin-poster.jpg">     
            <source src="http://zippy.gfycat.com/ElasticSameAmericanrobin.webm" type="video/webm">
            <source src="http://zippy.gfycat.com/ElasticSameAmericanrobin.mp4" type="video/mp4">
 </video> 

Pretty neat, in my opinion!

If you have any questions, comments, or suggestions, you're welcome to leave a comment below.


_You're awesome for reading this! Feel free to follow me on [GitHub](https://github.com/nelson-liu) or [subscribe to blog updates via email](http://eepurl.com/b8NcW5)._
