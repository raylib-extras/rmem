# rmem

A set portable memory structures to be used with any engine/framework.

By Kevin 'Assyrianic' Yonan ([@assyrianic](https://github.com/assyrianic))

rmem implements 3 types of memory structures:

 - [Memory Pool](memory-pool)
 - [Object Pool](object-pool)
 - [Double-Ended aka Bi(furcated) Stack](bistack)

## Memory Pool
	
The Memory Pool is a quick, efficient, and minimal free list & stack-based allocator.

Memory Pool's purpose is the following list:

* A quicker, efficient memory allocator alternative to `malloc` and friends.
* Reduce the possibilities of memory leaks for beginner developers.
* Being able to flexibly range check memory if necessary.

### Data Implementation

The memory pool encapsulates two public structs:

* `freeList` which is an abstracted doubly linked list consisting of a `head` and `tail` pointer to `MemNode`
* A `len` that tracks the amount of nodes the linked list holds.
* an array of doubly linked lists that store fixed size blocks called `buckets`.
```c
	typedef struct MemNode {
		size_t size;
		MemNode *next, *prev;
	} MemNode;

	typedef struct AllocList {
		MemNode *head, *tail;
		size_t len;
	} AllocList;
```
	
`arena` which is an array-based buffer consisting of...
* a byte pointer to the entire array `mem`
* a base pointer `offs` which changes position when allocating, deallocating, or defragging,
* and `size` which tracks how large the entire buffer is.
```c
	typedef struct Arena {
		uintptr_t mem, offs;
		size_t size;
	} Arena;
```
	
```c
	typedef struct MemPool {
		AllocList large, buckets[MEMPOOL_BUCKET_SIZE];
		Arena arena;
	} MemPool;
```

### Usage

The memory pool is designed to be used as a direct object.

We have two constructor functions:
```c
	MemPool CreateMemPool(size_t bytes);
	MemPool CreateMemPoolFromBuffer(void *buf, size_t bytes);
```
To which you create a `MemPool` instance and give the function a max amount of memory you wish or require for your data.
Remember not to exceed that memory amount or the allocation functions of the allocator will give you a NULL pointer.
	
So we create a pool that will malloc 10K bytes.
```c
	MemPool pool = CreateMemPool(10000);
```
	
When we finish using the pool's memory, we clean it up by using `DestroyMemPool`.
```c
	DestroyMemPool(&pool);
```

Alternatively, if you're not in a position to use any kind of dynamic allocation from the operating system, you have the option to utilize an existing buffer as memory for the mempool:
```c
	char mem[64000];
	MemPool pool = CreateMemPoolFromBuffer(mem, sizeof mem);
```
	
To allocate from the pool, we have two functions:
```c
	void *MemPoolAlloc(MemPool *mempool, size_t bytes);
	void *MemPoolRealloc(MemPool *mempool, void *ptr, size_t bytes);
```

`MemPoolAlloc` returns a (zeroed) pointer to a memory block.
```c
	// allocate an int pointer.
	int *i = MemPoolAlloc(&pool, sizeof *i);
```
	
`MemPoolRealloc` works similar but it takes an existing pointers and resizes its data. Please note that if you resize a smaller size, the data WILL BE TRUNCATED/CUT OFF.
If the `ptr` argument for `MemPoolRealloc` is `NULL`, it will work just like a call to `MemPoolAlloc`.
```c
	// allocate an int pointer.
	int *i = MemPoolRealloc(&pool, NULL, sizeof *i);
	
	// resize the pointer into an int array of 10 cells!
	i = MemPoolRealloc(&pool, i, sizeof *i * 10);
```
	
To deallocate memory back to the pool, there's also two functions:
```c
	void MemPoolFree(MemPool *mempool, void *ptr);
	void MemPoolCleanUp(MemPool *mempool, void **ptrref);
```
	
`MemPoolFree` will deallocate the pointer data back to the memory pool.
```c
	// i is now deallocated! Remember that i is NOT NULL!
	MemPoolFree(&pool, i);
```
	
`MemPoolCleanUp` instead takes a pointer to an allocated pointer and then calls `MemPoolFree` for that pointer and then sets it to NULL.
```c
	// deallocates i and sets the pointer to NULL.
	MemPoolCleanUp(&pool, (void **)&i);
	// i is now NULL.
```
	
Using `MemPoolCleanUp` is basically a shorthand way of doing this code:
```c
	// i is now deallocated! Remember that i is NOT NULL!
	MemPoolFree(&pool, i);
	
	// i is now NULL obviously.
	i = NULL;
```
	
If there's a moment when you want to not only free all your allocated data but refresh the allocator in its entirety, there's:

```c
	void MemPoolReset(MemPool *mempool);
```
which will completely re-merge all freelist blocks and zero out the allocator's buffer memory.
Make sure to `NULL` all living pointers when doing this.

Finally, to get the total amount of memory remaining (to make sure you don't accidentally over-allocate), you utilize this function:
```c
	size_t GetMemPoolFreeMemory(const MemPool mempool);
```

## Object Pool

The Object Pool is a fast and minimal fixed-size allocator.

Object Pool was created as a complement to the Memory Pool.
Due to the general purpose nature of Memory Pool, memory block fragmentations can affect allocation and deallocation speeds. Because of this, the Object pool succeeds by having no fragmentation and accommodating for allocating fixed-size data while the memory pool accommodates allocating variadic/differently sized data.

### Data Implementation

The object pool is implemented as a hybrid array-stack of cells that are large enough to hold the size of your data at initialization:
```c
	typedef struct ObjPool {
		uintptr_t mem, offs;
		size_t objSize, freeBlocks, memSize;
	} ObjPool;
```

### Usage

The object pool is designed to be used as a direct object.
We have two constructor functions:
```c
	ObjPool CreateObjPool(size_t objsize, size_t len);
	ObjPool CreateObjPoolFromBuffer(void *buf, size_t objsize, size_t len);
```
	
To which you create a `ObjPool` instance and give the size of your object and how many objects for the pool to hold.
So assume we have a vector struct like:
```c
	typedef struct vec3D {
		float x,y,z;
	} vec3D_t;
```
... which will have a size of 12 bytes.
	
Now let's create a pool of 3D vectors that holds about 100 3D vectors.
```c
	ObjPool vector_pool = CreateObjPool(sizeof(vec3D_t), 100);
```
	
Alternatively, if for any reason that you cannot use dynamic memory allocation, you have the option of using an existing buffer for the object pool:
```c
	vec3D_t vectors[100];
	ObjPool vector_pool = CreateObjPoolFromBuffer(vectors, sizeof(vec3D_t), 1[&vector] - 0[&vector]);
```
The buffer MUST be aligned to the size of `size_t` AND the object size must not be smaller than a `size_t` either.
	
Next, we start our operations by allocating which will always allocate ONE object...
If you need to allocate something like an array of these objects, then you'll have to make an object pool for the array of objects or use Memory Pool.

Allocation is very simple nonetheless!
```c
	vec3D_t *origin = ObjPoolAlloc(&vector_pool);
	origin->x = -0.5f;
	origin->y = +0.5f;
	origin->z = 0.f;
```
	
Deallocation itself is also very simple. There's two deallocation functions available:
```c
	void ObjPoolFree(ObjPool *objpool, void *ptr);
	void ObjPoolCleanUp(ObjPool *objpool, void **ptrref);
```
	
`ObjPoolFree` will deallocate the object pointer data back to the memory pool.
```c
	ObjPoolFree(&vector_pool, origin);
```
	
Like Memory Pool, the Object Pool also comes with a convenient clean up function that takes a pointer to an allocated pointer, frees it, and sets the pointer to NULL for you!
```c
	ObjPoolCleanUp(&vector_pool, (void **)&origin);
```
	
Which of course is equivalent to:
```c
	ObjPoolFree(&vector_pool, origin), origin = NULL;
```


Once you're free with the object pool, you dispose of it by using `DestroyObjPool`:
```c
	DestroyObjPool(&vector_pool);
```

## BiStack

The BiStack is a fast & efficient bifurcated stack "bi-stack" allocator.

BiStack's purpose is the following list:

* A quick, efficient way of allocating temporary, dynamically-sizeable memory during various operations.
* Bifurcated to allow certain temporary data to have a different lifetime from other, temporary data.

### Data Implementation

The bifurcated stack encapsulates one public struct:
* `mem` which is an unsigned integer large enough to represent a pointer; the pointer value it holds is the memory address of the backing buffer.
* `front` holds a pointer value representing the first aka _front_ portion of the bifurcated stack.
* `back` holds a pointer value representing the second aka _back_ portion of the bifurcated stack.
* and a `size` that holds the amount of bytes of the backing buffer.
```c
	typedef struct BiStack {
		uintptr_t mem, front, back;
		size_t size;
	} BiStack;
```

### Usage

The bi-stack is designed to be used as a direct object.
There are two constructor functions:
```c
	BiStack CreateBiStack(size_t len);
	BiStack CreateBiStackFromBuffer(void *buf, size_t len);
```
To which you create a `BiStack` instance and give the function a max amount of memory you wish or require for your data.
Remember not to exceed that memory amount or the allocation functions of the allocator will give you a NULL pointer.

So we create a bistack that will malloc an internal buffer of 10K bytes.
```c
	BiStack bistack = CreateBiStack(10000);
```

When the bi-stack is no longer needed, we clean it up by using `DestroyBiStack`.
```c
	DestroyBiStack(&bistack);
```

Alternatively, if you're not in a position to use any kind of dynamic allocation from the operating system, you have the option to utilize an existing buffer as memory for the bistack:
```c
	char mem[64000];
	BiStack pool = CreateBiStackFromBuffer(mem, sizeof mem);
```

To allocate from the bistack, we have two functions:
```c
	void *BiStackAllocFront(BiStack *destack, size_t len);
	void *BiStackAllocBack(BiStack *destack, size_t len);
```

**NOTE**: _The two allocator functions do **NOT** zero memory._
```c
	// allocate an int pointer from the front.
	int *i = BiStackAllocFront(&bistack, sizeof *i);

	// allocate a float pointer from the back.
	float *f = BiStackAllocBack(&bistack, sizeof *f);
```

Unlike the other allocators, the Bi-stack does not require you free given pointers back to the allocator. However, you will need to reset the bi-stack in order to "free" the data back to the bi-stack. Here are three functions that resets the bi-stack:
```c
	void BiStackResetFront(BiStack *destack);
	void BiStackResetBack(BiStack *destack);
	void BiStackResetAll(BiStack *destack);
```
`BiStackResetFront` will reset allocation data for the front portion of the bi-stack. So ALL pointers given from the front portion can be overwritten if any new pointers allocated also point to those parts. Same deal for `BiStackResetBack` and the back portion of the bi-stack.

If both portions need to be reset, `BiStackResetAll` will do the job.

An important caveat with the bi-stack is that if the back portion or the front portion collide with one another, you will be given a `NULL` pointer. To check if they will collide, `intptr_t BiStackMargins(BiStack destack);` is used to get the difference between how far, in bytes, the back portion is from the front.

The way this works is that, when allocating from the front portion, the `front` member is increased. When allocating the back portion, the `back` member is decreased. If the front portion reaches the back, that means the front portion of the bi-stack is out of memory, same deal for the back portion if it reaches the front. Thus, `BiStackMargins` provides you the difference between these two portions; a number of 0 or less means that one of the portions has reached the other and a reset is necessary.

## License

`rmem` is licensed under an unmodified zlib/libpng license, which is an OSI-certified, BSD-like license that allows static linking with closed source software. Check [LICENSE](LICENSE) for further details.
