# PA6 – Malloc and Free

Your task on this assignment will be to implement (a version of) `malloc` and `free`, along with some auxiliary functions.

Github Classroom link:

https://classroom.github.com/a/mvSj8_D5

It is due at 11:59 PM on Tuesday, December 5.

## Functions you Will Implement

You will implement 6 functions for this assignment (which may have helpers).

```
void* my_malloc(size_t size);
int my_free(void* ptr);

void consolidate();
void print_heap();
int free_space();
int live_data();
```

These headers, along with some global data that you are required to use, are provided in `alloc.h`.

## Heap Layout

The heap should be laid out using _memory entries_, which have a specific format.

A _memory entry_ is either a free entry or an occupied entry.

A free entry contains:

- A _size_ in the first four bytes that indicates how many free bytes follow the given word. The size must always be a multiple of four
- A _next pointer_ in the next four bytes (possibly `NULL`) that holds the address of the next free entry in the heap
- Arbitrary values in the bytes after these 8 initial bytes

An occupied entry contains:

- A _size_ in the first four bytes that indicate how many bytes are occupied after it, *plus one*. This allows us to tell the difference
between occupied and free entries – if the size ends in a 1, we know its occupied, if it ends in 0, we know it's free.
- Arbitrary values (used by the user, not by our library, while the memory is in use) in the bytes following this word.

Every entry must be at least 8 bytes, counting the first word for the size.

## Global Information

Aside from the information in the heap, your implementation will keep track of:

- a pointer, the _current free pointer_, that stores the address of the most-recently-freed entry on the heap
- a pointer to the beginning of the heap

These will be provided as global variables; you should use the helpers
`setup_heap` and `teardown_heap`, which initialize them, at the beginning and end of each test.

## `void* my_malloc(size_t size)`

`my_malloc` should traverse the free list from the beginning until it finds a free
entry that is large enough for the requested size. It should create a new
occupied entry at that location. There are a few cases to consider:

- Splitting a large free entry: If the size requested is 8 or more bytes
  _smaller_ than the free entry, it should be split. The first part should be
  used for the occupied entry, and the second should become a smaller free
  entry. The incoming link to the free entry should be updated to refer to its
  new location, the size of the free entry should be updated to reflect the
  appropriate amount of space, and the outgoing link from the free entry should
  be maintained.
- Re-using the whole free entry: If the size requested is within 4 bytes of the
  size of the free entry, the whole entry should be re-used for the allocation.
  This means that any incoming links need to be changed to be forwarded past
  this node, similar to removing a node from a linked list.

In both cases the address returned should be to the word immediately _after_
the size metadata for the new occupied block.

### Running out of Memory and Consolidating Free Memory

In a call to `my_malloc`, it should first traverse the free list to find a large
enough free entry that the specified `size` can fit in. If such a large enough
free entry isn't found, it should return `NULL` and not change anything.

**NOTE**: A clever implementation could try calling `consolidate` and then
searching again. To keep `my_malloc` simple, you should _not_ do this in your
implementation.

## `int my_free(void* ptr)`

`my_free` should:

- Change the metadata of the entry referred to by the given pointer so it
  indicates a free entry (this just means changing the least-significant bit
  from 1 to 0)
- Make the free entry's next pointer refer to the current start of the free list
- Make the current start of the free list refer to this newly-created free entry

It should return 1 for success, and 0 for failure as indicated below in
**Double Fre** and **Invalid Address Free**.

## Errors and Boundary Cases for `my_malloc` and `my_free`

**Full Memory**: If there is not enough memory free in a call to `my_malloc`,
`my_malloc` should return `NULL` and not add any new occupied entries to the
heap.

**Odd Sizes**: `my_malloc` should always allocate memory in multiples of 4, and
should never allocate less than 4 bytes. You should make sure if the user
passes in a size that violates this, you round *up* to an appropriate amount.
If the size is non-positive, `my_malloc` should return NULL.

**Allocating Negative or Zero Bytes**: If the user attempts to allocate zero or
fewer bytes, `my_malloc` should return `NULL`.

**Double Free**: If the program calls `my_free` with a pointer to memory that
contains a free entry, `my_free` should change nothing and return `0`.

**Invalid Address Free**: If `my_free` is called with an address that isn't a
multiple of 4, it can't possibly have been allocated by `my_malloc`.
Similarly, if `my_free` is called with an address that isn't in the range of
the provided heap, it can't have been allocated with `my_malloc`. In both
of these invalid cases, `my_free` should change nothing and return 0.

## `void print_heap()`

This function is optional, it prints the layout of your heap structure so that
you can use to verify your implementation. You might find enhancements to the
version provided here:

https://github.com/ucsd-cse30-f17/lectures/blob/master/11-27-lec24/slow-malloc.c#L101

to be helpful.

## `int free_space()`

This function returns the total amount of free space on the heap. You should
iterate over the free list and sum up all the sizes. Don't consider metadata in
the count, and don't consider consolidation, just process the heap as it is.

## `int live_data()`

This function returns the total amount of occupied space on the heap. You
should iterate the entire heap and sum up all the sizes of occupied entries.
Don't consider metadata in the count.

Note that this function effectively detects memory leaks! If `live_data()` is
anything other than 0 at the end of a program, that means some memory was left
un-freed. This is a strong hint about how `valgrind` might be working under the
hood! It replaces the default implementations of `malloc` and `free` with its
own, and does extra bookkeeping and checks for live data at the end of the
program.

## `void consolidate()`

This function should _consolidate_ (the readings use _coalesce_) adjacent free
blocks. It should traverse the entire heap, starting at the earliest address
(e.g. using `heap_ptr`, ignoring the free list pointer), and look for adjacent
free blocks to merge.

After it is complete, the following invariants should hold:

- Each `next` pointer from a free entry points to a _higher_ address than where
  it is stored.
- The `current_free_list` pointer holds the address of the free entry that is
  _lowest_ in memory.
- There are no longer any adjacent free entries in memory

That is, the `current_free_list` pointer will refer to the “first” free entry
(reading forwards from the start of the heap), and each pointer for a next free
entry will point “forward” in the heap. This may significantly reorganize the
structure of the free list!

While this sounds like a constraint, it actually makes the consolidation
algorithm more straightforward. A hint is to think about an algorithm that
keeps track of a current free entry that is “waiting” to have its next pointer
set. When the next non-adjacent free entry is found, that entry's pointer can
be set to point to the just-found free entry.

## Example


Here's a worked example in images that shows many of the interesting cases
you'll encounter; turning this into a test that works is a great first
milestone.

[Example](EXAMPLE.md)


## Worklist


1. First implement `my_malloc` in the simplest way possible: put the size
information into the first word, and update the current free list pointer. Have
`my_free` do nothing. Make sure you can test that the correct memory is set.
2. Then, work on `print_heap`, making sure you can usefully display the values
on your heap for quick debugging.
3. Next, start working on `my_free`, and add cases to `my_malloc` incrementally
with tests to handle manipulating the free list pointers, and splitting free
entries. 
4. Then, work on `free_space()` and `live_data()`
5. Implement `consolidate`
6. Think about the README questions while working through the above; fill them
and their workloads in last.

## README/Open-ended Questions

`my_malloc` and `my_free` behave differently on different workloads. You should
design some tests that stress the implementation in different ways. By
“workload” we mean a sequence of calls to `my_malloc` and `my_free`. You might use
loops and helper functions to create a particular workload. For both, put the
written answer in your README and refer to the tests you wrote in test.c.

1. Design a workload that causes lots of *fragmentation*. That is, you should
get a `NULL` response from `my_malloc` for a relatively small request while
`free_space` reports a relatively large amount of free space. Provide it as a
test function in your tests file, and justify it with 3-4 sentences.

2. Design a workload that processes `my_malloc` calls slowly, but would be faster
if the order of the free list was reversed (that is, if we _appended_, rather
than _prepended_, the freed nodes onto the list). Provide it as a test function
in your tests file, and justify it with 3-4 sentences.


## Grading

- 20% `my_malloc`
- 20% `my_free`
- 10% `consolidate`
- 5% `live_data`
- 5% `free_space`
- 10% Quality of testing
- 25% README
- 5% Overall Style
