## mmap vs read

* A call to mmap has more overhead than read (just like epoll has more overhead than poll, which has more overhead than read). Changing virtual memory mappings is a quite expensive operation on some processors for the same reasons that switching between different processes is expensive.
* The IO system can already use the disk cache, so if you read a file, you'll hit the cache or miss it no matter what method you use.

However,
* Memory maps are generally faster for random access, especially if your access patterns are sparse and unpredictable.
* Reading a file directly is very simple and fast.

The discussion of mmap/read reminds me of two other performance discussions:

* Some Java programmers were shocked to discover that nonblocking I/O is often slower than blocking I/O, which made perfect sense if you know that nonblocking I/O requires making more syscalls.

* Some other network programmers were shocked to learn that epoll is often slower than poll, which makes perfect sense if you know that managing epoll requires making more syscalls.

**Conclusion**: Use memory maps if you access data randomly, keep it around for a long time, or if you know you can share it with other processes (MAP_SHARED isn't very interesting if there is no actual sharing). Read files normally if you access data sequentially or discard it after reading. And if either method makes your program less complex, do that. For many real world cases there's no sure way to show one is faster without testing your actual application and NOT a benchmark.

## refer
* [https://stackoverflow.com/questions/45972/mmap-vs-reading-blocks](https://stackoverflow.com/questions/45972/mmap-vs-reading-blocks)