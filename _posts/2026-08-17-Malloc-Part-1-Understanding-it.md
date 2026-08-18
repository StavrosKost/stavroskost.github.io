# Part 1: Malloc Internals

Hello everyone today i will talk about how malloc works and my understandind of its internals and on the next parts i will build my own malloc as well as do some exploitation on malloc from glibc.

So instead of going like normal that malloc works like this and like that, i will begin on how i thought malloc worked which turned out completely wrong and then take you step by step.

A lot of my images will be taken from [Azeria Labs](https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/) so please check them out as well.

---

So initially i thought that the program will ask the kernel repeatedly for some memory meaning, when the user did malloc(10), the program will ask kernel for 10, it will fetch it and boom it has memory, althought this seemed to me like a good solution it has a lot of problems. 

Firstly, asking the kernel repeatedly to give memory is bad practice as well as a waste of resources. When calling the kernel you need to firstly occupy some resources because you ask it to fetch memory, then fetch the memory and then get the memory. 

Instead the solution is more elegant, the program asks a large amount of memory, the actual default size that glibc uses in my knowledge is 0x21000 bytes and then everything is about managing that memory in a smart way. 

---

## Basics

Lets start with the basics, there are 2 important fixed sizes, this is on x64

### Page size
Page size is 4KB = 4096 bytes
This is fixed and it is the smallest unit of data for memory management and it is used in [virtual memory](https://en.wikipedia.org/wiki/Virtual_memory)

### Alignment
When you are allocating some space whether you like it or not you get your number but in multiples of 16 which is the minimum alignment, meaning if you ask malloc(10), which says give me 10 bytes, the kernel will give you 16 which is the closest upper multiple of 16, this does not work in lower, meaning if you ask malloc(17) it will not allocate 16, it will instead allocate (32).

### Heap Memory Organization
The heap memory is actaully organized as ARENAS, which is essentially what i said it does in the beginning, a giant block of memory and there can be multiple arenas this will be discussed later in this article. 

## Dive deeper

So what is that smart way that memory is managed and why did it take so long like hurry up. 

Well how do we start, ahh yes from here

So the way they figured on how to connect the blocks of data that they want is via double linked lists, which contains some metadata and also the data that the program/user wants to insert. Both the [malloc chunk and free](https://github.com/lattera/glibc/blob/master/malloc/malloc.c) can be found on the link on line 1059 on this current date

### Malloc Chunk Metadata
```
size of previous chunk
size of chunk in bytes |A|M|P| (arena,mmapped, previous in use)
forward pointer to next chunk (these are used only for free)
backwards pointer to previous (--//--)
//only important for freeing blocks that are send to large bin
forward size pointer to next chunk (--//--)
backwards pointer to previous (--//--)
```
### Free Metadata
```
size of previous chunk
size of chunk in bytes |A|0|P|
forward pointer to next chunk in list
back pointer to previous chunk in list
unused space
size of chunk used for application data
size of next chunk in bytes
```

Lets explain this a bit because this might be confusing to beginners

So for example if i have allocated 2 chunks and then free the first one then the second one in the size of chunk in bytes \|A\|0\|P\| P should be now 0 since it is no longer in use. So we can see it is really useful to check if the last bytes i 0 or 1 ie if it is 0x00000021 the previous chunk is in use if it is 0x00000020 the previous chunk is no longer in use

\|A\| If i it 1 chunk came from mmap and if it is 0 it came from main arena and the main heap

\|M\| Chunk in allocated with mmap call and is not part of a heap

---

![Azure labs](/img/chunk-freed-CS.png)
From azure labs

---

So a problem that some of you might see is this
```
int *p1 = malloc(10);
int *p2 = malloc(20);
free(p1);
free(p2);
int *p3 = malloc(30);
```
If you can't see the problems that fine or is it????? I am joking :)

The problem is that even thought we have freed 30 bytes, the program will try to fit inside the 10 the 30 and say it does not fit, then again with the 20, it will see that it will not fit and go find another spot even thought there are 30 bytes free there, SOLUTION????(hint see that image)

Well the solution is coalescing, meaning if will see that the previous chunk is free and join them into one bigger freed chunk of 30 bytes inside of 2 smaller chunks.

Pretty simple huh? Well it gets better there is another problem called [memory fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing)), which is essentially when the memory allocation probides more space than needed causing waste of free space. Example because examples are the best

```
p1 = malloc(0x70)
p2 = malloc(0x30)
p3 = malloc(0x90)
free(p1)
free(p2)
free(p3)
p4 = malloc(0x70)
p5 = mallco(0x90)
```

This makes that 0x30 is essentially pretty useless because it is very unlikely that we will need exactly 0x30 again instead of making a larger 0x130(this is correct and not 0x190 this is hexdecimal and not decimal). 

## So what do we do to fight fragmentation????

Well there are a few solutions, lets start with naming them:
Unsorted bin, small bin, large bin, fast bin, tcache and thats it for now 

Their order to operation is this:
```
TCACHE -> FAST BIN -> UNSORTED IN -> SMALL/LARGE BIN -> TOP CHUNK -> EXTEND HEAP
(thread)   (arena)
```

Lets explain each

## Small & Large bin

So what is a bin? 
A bin is a fixed size chunk of memory which makes it a lot easier to manage, the small bin contains chunk from size 32 up to 1024 chunk size and there are 62 of them

![smallbin](/img/small-bin.png)

---

Large bin are very similar to small bins but for larger sizes


![largebin](/img/large-bin.png)

---

## Unsorted Bin

This is a temporary holding area for freed chunks, it is used in order to speed up allocation and deallocation it is only 1 bin 

![unsorted-bin](/img/bins-unsorted.png)

## Fast bin

Fast bins are single linked lists because we want them to be faster, also very important the in use bit needs to be 1 because if it is 0 it will be consolidated and we don't want that

This receives freed chunks before the unsorted bin, if the size does not match it then goes to unsorted bin, we have 10 16 bytes fast bins, which includes data + metadata so probably closer to 5, since 2 16 bytes are needed for data + metadata, in the below image the first 16 bytes are the metadata, we can see the 0x21 which is the size and then a lot of 0xaa, which is the data itself 

![size](/img/Pasted%20image%2020260729093104.png)

## Before we go to TCACHE lets explain Heap ARENAS more

The difference between main_arena and arenas like arena1, arena2 etc is that main_arena is created using sbrk and arena1,arena2 are created using mmap and every arena is assigned to a thread. In 64 bit systems, the max cores is calculated using n* sizeof(long) where long = 8 so the max is 128 arenas if we have a 16 core processor

Firstly we have arena lock, in order to prevent race condition and heap corruption a mutex is placed. Lock mutex has a large overhead and each time it is called it requires a syscall.

Mmap created arenas that are placed after the main_arena lets call them arena1,arena2 etc. This will create new heap. Each arena points to the next arena, and loops around meaning the last arena->next points to the main_arena

## TCACHE

It adds some performace gain in multi-threading application using the heap.
When we have multiple threads, all of these would normally content the main_arena using a lock(mutex). To avoid the congestion we use multiple arenas and each thread tries to access 1 arena, sometimes a thread might need to find data in another arena which is handled by another thread, so the race condition prevented is the allocation and free. 
Here is where Tcache somes, it is a cache that has blocks of memory it would be provided to a thread in a **LOCKLESS** manner
It is similar to fast bin but for multi threading


That is it for part 1, i hope i did not tire you, i am planning on releasing 2 more parts as stated in the beginning so stay tuned and curious. See ya!

