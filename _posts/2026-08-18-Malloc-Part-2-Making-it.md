# Memory Allocator

This was a project of mine that i recently created and it sort of mimics the GLIBC allocator but it is a lot simpler. Check it [here](https://github.com/StavrosKost/custom-c-allocator) if you want.

This is going to be the journey of how i made it.

## Beginning

So first step is to understand what you need to do, how do i request space from kernel? How do i manage it? Do i have multiple structures?

So when you try to malloc waht do you do?
You do malloc(10) for example
But what does this do, what does this return etc?

Well the base form is actaully this
```
void *malloc(size_t size);
```

So essentaily malloc returns a pointer to the address that will hold your data + metadata and takes an input size in order to know how much you need.

There are 2 ways to ask the kernel for memory [sbrk](https://linux.die.net/man/2/sbrk) and [mmap](https://linux.die.net/man/3/mmap). 

What is their difference?? Sbrk existed before the mmap if someone is using earlier versions of linux but probably irrelevant for modern linux. So main difference is that sbrk extends program break of the continues memory map of the heap while the mmap creates an independant and autonomous part of virtual memory.

So for this reason we will use sbrk because we don't want to create an independant part of virtual memory.

So a simple way to do it and i will explain exactly what this does is below:

```
void *malloc(size_t size){
    void *p = sbrk(0);
    void *request = sbrk(size);
    if (request == (void*) -1){
        return NULL;
    }else{
        assert(p == request);
        return p;
    }
}
```

Essentially, sbrk(0) returns the current addres of the program break, then we request the program break to move by size in order to gain that memory and check if the sbrk returned normal or if it failed, then we need to asser that the current address is the address after the extension and then return that address

Then we have free, which frees the memory from the heap and it is 
```
void free(void *ptr);
```

Essentially it will take the address and try to free it, for the chunks i will use the same struct that the glibc malloc uses

```
struct malloc_chunk{
    size_t size_previous_chunk;
    size_t size;
    struct malloc_chunk* next;
    struct malloc_chunk* previous;
    char data[];
};
```

This struct has the size of the previous chunk, the size of the current chunk, the pointer to the next and previous chunk as well as the data

Almost everything up to now was taken by this [tutorial](https://danluu.com/malloc-tutorial/) which was extremely helpful with the beginning of how malloc worked but time to improve

## My implementations

### Malloc

So to improve the model, i decided to try to use a way to reduce memory fragmentation, i decided on small bins. Lets me show you my code and explain slowly what each does

```
struct malloc_chunk *request_space(size_t size){
    struct malloc_chunk *block;
    block = sbrk(0);
    void *request = sbrk(size);
    assert((void*)block == request);
    if (request==(void*)-1){
        return NULL;
    }
    block->next = NULL;
    size = size | 0x01;
    block->size = size; 
    global_tail = block;
    return block;
}
```

So this is a function that will request space from the kernel similarly to the tutorial i will use sbrk and assert their are equal and it did not fail, since we are requesting space the next block will definetely be empty so we initialize it with NULL, i used the technique that instead of if the last nibble of the size is 1 that previous chunk is in use, i did it so that the current chunk is in use. And i also have a global_tail as well as a head(not shown here) which keeps track of the linked list start and end. 

The reason i decided to explain this first is because it was used inside my malloc and in order to make sense i had to explain it so lets proceed

```
void *malloc(size_t size){
    struct malloc_chunk *chunk;
    if (size<=0){
        return NULL;
    }
    //this make is 16 byte aligned
    while((size+META_SIZE)%16!=0){
        size++;
    }
    size = size + META_SIZE;
    if (!global_base){
        chunk = request_space(size);
        if (!chunk){
            return NULL;
        }
        global_base = chunk;
        global_tail = global_base;
    }else{
        struct malloc_chunk *last = global_tail;
        chunk = from_bin(size);
        if (!chunk){
            chunk = request_space(size);
            if (!chunk){
                return NULL;
            }
        }
    }
    
    return (chunk+1);
}
```

This malloc firstly will check if the size is greater than 0 since it is negative of zero there is no need for malloc to happen so we just return NULL, then we do a size alignment so that the meta data + the data will be in multiples of 16 is very helpful since if we inspect the memory we will be able to tell where each chunk starts and finishes. Then we check if there is a global_base meaning if this is the first malloc or not, if it is then we request_space as shown above if it returned correctly then we initialize the global base and tail else we return NULL

If it is not the first malloc, then we say that the last chunk was the global_tail then we check if there is space inside the bin this is not shown yet i will explain it right after, if there is no space available inside the bin then we request space, and we return chunk+1. Why +1? You may ask well +1 justs jumps the metadata and goes straight inside the data so that is the reason.

So what is from_bin?

```
struct malloc_chunk *from_bin(size_t size){
    size_t i = 0;
    struct malloc_chunk* tmp;
    if (size>=1024){
        return NULL;
    }
    i = (size-32)/16;
    if (smallbin[i]==0){
        return NULL;
    }
    tmp = smallbin[i];
    smallbin[i] = tmp->next;
    tmp->size |= 0x1;
    return tmp; 
}
```

Firstly as i said on the previous post the small bin goes up to 1024 bytes so thats why the size is up to 1023 in this case, then i is the index that points to the corresponding bin, it is simple math, based on the size we subtract 32 since the smallest size is 32 from data + metadata and then divide by 16 to point to the bin 0 up to 63. Then if it is 0 it means it is in use or unitialized. Then we make it so that the smallbin[i] will point to the next available chunk in bin because it is hard to explain let me show you
![small](/img/small-bin.png)

So lets say someone wants to allocate 16 bytes, it will go to smallbin[0] and use the first chunk, if he wants again 16 bytes then it will go to the next available inside smallbin[0]. 

And after that it will make the last nibble of the tmp->size 1 to indicate it is in use and then return it

### Free

So my implementation of free is pretty simple

```
void free(void *ptr){
    if (!ptr){
        return;
    }

    struct malloc_chunk* control = (struct malloc_chunk*)((char*)ptr - META_SIZE); //ptr points to the given malloc_chunk
    control->size = control->size & ~0x0f; //the block size will have 0 in the last nibble of the size to indicate that it is not in use
    to_bin(control, control->size);
}
```
So we check if there is a pointer else we return, then we take the the control chunk which is the address which points to the data and subtract the meta_size to point to the beginning of the chunk and NOT the beginning of the data. Then we zero out the last nibble of size and then update the bin 

```
struct malloc_chunk *to_bin(struct malloc_chunk* control,size_t size){
    if(!control || size>=1024){
        return NULL;
    }
    size_t i = 0;
    i = (size-32)/16;
    control->next = smallbin[i];
    smallbin[i] = control;
    return smallbin[i]; 
}
```

Which also checks the size and if there is a pointer, then go to the correct index based on the size then the next will point back to the smallbin[i] and the smallbin[i] will be the control essentailly freeing it and then returning it

And thats it here is my code complete:

```
#include <assert.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>

struct malloc_chunk{
    size_t size_previous_chunk;
    size_t size;
    struct malloc_chunk* next;
    struct malloc_chunk* previous;
    char data[];
};

#define META_SIZE sizeof(struct malloc_chunk)

void *global_base = NULL;
void *global_tail = NULL;

struct malloc_chunk* smallbin[62];
struct malloc_chunk *request_space(size_t size);
struct malloc_chunk *to_bin(struct malloc_chunk* control,size_t size);
struct malloc_chunk *from_bin(size_t size);

void free(void *ptr);

void *malloc(size_t size){
    struct malloc_chunk *chunk;
    if (size<=0){
        return NULL;
    }
    //this make is 16 byte aligned
    while((size+META_SIZE)%16!=0){
        size++;
    }
    size = size + META_SIZE;
    if (!global_base){
        chunk = request_space(size);
        if (!chunk){
            return NULL;
        }
        global_base = chunk;
        global_tail = global_base;
    }else{
        struct malloc_chunk *last = global_tail;
        chunk = from_bin(size);
        if (!chunk){
            chunk = request_space(size);
            if (!chunk){
                return NULL;
            }
        }
    }
    
    return (chunk+1);
}

void *calloc(size_t nelem, size_t elsize){
    size_t size = nelem * elsize;
    void *ptr = malloc(size);
    if (ptr){
        memset(ptr, 0 ,size);
    }
    return ptr;
}

void free(void *ptr){
    if (!ptr){
        return;
    }

    struct malloc_chunk* control = (struct malloc_chunk*)((char*)ptr - META_SIZE); //ptr points to the given malloc_chunk
    control->size = control->size & ~0x0f; //the block size will have 0 in the last nibble of the size to indicate that it is not in use
    to_bin(control, control->size);
}

struct malloc_chunk *to_bin(struct malloc_chunk* control,size_t size){
    if(!control || size>=1024){
        return NULL;
    }
    size_t i = 0;
    i = (size-32)/16;
    control->next = smallbin[i];
    smallbin[i] = control;
    return smallbin[i]; 
}

struct malloc_chunk *from_bin(size_t size){
    size_t i = 0;
    struct malloc_chunk* tmp;
    if (size>=1024){
        return NULL;
    }
    i = (size-32)/16;
    if (smallbin[i]==0){
        return NULL;
    }
    tmp = smallbin[i];
    smallbin[i] = tmp->next;
    tmp->size |= 0x1;
    return tmp; 
}

struct malloc_chunk *request_space(size_t size){
    struct malloc_chunk *block;
    block = sbrk(0);
    void *request = sbrk(size);
    assert((void*)block == request);
    if (request==(void*)-1){
        return NULL;
    }
    block->next = NULL;
    size = size | 0x01;
    block->size = size; 
    global_tail = block;
    return block;
}
```

Thanks for reading the next part might take a little longer since it will be the exploitation so stay tuned. See ya!