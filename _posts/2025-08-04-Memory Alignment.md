---
layout: post
title: "Memory Alignment"
tags: golang memory+management
---

Being a self taught programmer, naive me always thought that memory occupied by structs is equal to sum of memory occupied by individual fields in structs. Well... I was wrong. Moreover, the memory occupied can change if the order of fields change in the definition of structs. This was a more shocker for me.

Let's take a look at a sample go program to see this in action.
``` go
package main

import (
	"fmt"
	"unsafe"
)
type DemoStruct1 struct {
	a int8
	b float64
	c int32
}

// Point to note: order of fields is changed in DemoStruct2

type DemoStruct2 struct {
	a int8
	b int32
	c float64
}
func main() {
	d1 := DemoStruct1{
		a: 1,
		b: 1,
		c: 1,
	}
	fmt.Println("Size occupied by d1: ", unsafe.Sizeof(d1))

	d2 := DemoStruct2{
		a: 1,
		b: 1,
		c: 1,
	}
	fmt.Println("Size occupied by d2: ", unsafe.Sizeof(d2))
}
```
The total sum of fields in the defined structs is 11 bytes. However, the output of this program is
```
Size occupied by d1:  24
Size occupied by d2:  16
```
So, why does this happen?

The answer lies in optimizations around memory access.

#### Background

In the evolution of computers, the processor speeds saw massive performance scale. Unfortunately, that was not the case for memory. Result - reading and writing to memory became a costly operations in terms of CPU cycles.

To tackle this, lot of optimizations were developed. There is a hierarchy of caches to reduce the memory calls. These are typically called L1, L2 and L3 caches. Cache at each level is faster in access than the cache in level below. However, the size of cache gets smaller as we go up the level. Based on the frequency of the access, the data is pushed up the layer.

There's one more fundamental optimization that's put in place. A memory transaction happens not for individual bytes but for bunch of bytes together. This means whenever CPU wants read or write data to the memory, instead of doing it byte by byte, a block of bytes are read or written simultaneously. The number of bytes involved is dependent on CPU architecture. This is typically called a *cache line* or a *cache block*. Working with cache line introduces necessity to align the memory of the data that is stored in these caches.

#### What is memory alignment
Generally, it is said that the addresses of the elements stored in memory are *aligned to* a number. This basically means that the memory addresses are completely divisible by that number. This number is dependent on the cpu architecture and is decided by the language compiler. e.g. when one says addresses are aligned to 64, that means, the memory addresses for elements will be (or in other words, bytes of the elements will start from addresses) 0, 64, 128 and so on. To properly align the elements, the compiler adds additional empty bytes to individual elements of structs and may add some bytes at the end of the struct as well. This is called as *padding*.

#### Why is memory alignment crucial
Let's see what happens when memory addresses are not aligned correctly according to cache line size. Assume a 32 bit (4 bytes) integer is stored at address 126 on a system where cache line size 64 bytes. The integer bytes will then spread from 126 till 129. To completely read the integer, the processor will need to issue two memory reads. First, for the location 64 (which read bytes from 64-127) and the second for the location 128 (which will read from 128-191). So, a misalignment causes an extra memory read resulting in time delays.

On the other hand, if the integer was stored from byte 128, the CPU will need to read the cache line starting from 128 only saving the extra read.

#### A side effect of cache lines
An interesting side effect is seen due to cache lines in case of multicore environments. Lets say that two integers, 'i' and 'j' reside in the same cache line.
Core 1 modifies integer 'i'. The value is updated in Core 1's cache (L1 cache). However, that particular cache line is marked as dirty. Now, let's say, another core, say Core 2, wants to access integer 'j'. Since the cache line in which 'j' resides is marked dirty, Core 2 cannot just retrieve 'j' from it's own cache and it will need to get it from higher (shared) cache. This is results in more access time. This is called as *false sharing*.

#### Memory alignment in action
Now, we know how structs occupy more memory due to padding. To understand exactly how many bytes are padded by the compiler, we need to refer Go spec's section on [Size and Alignment guarantees.](https://go.dev/ref/spec#Size_and_alignment_guarantees) Below is the size guarantee for fields in structs defined in our example.

| **Type**       |** Size (in bytes)**|
| :------------- | :-------------:    |
| int8           | 1                  |
| int32          | 4                  |
| float64        | 8                  |

The go spec makes following alignment guarantees:
>
> - For a variable x of any type: unsafe.Alignof(x) is at least 1.
> - For a variable x of struct type: unsafe.Alignof(x) is the largest of all the values unsafe.Alignof(x.f) for each field f of x, but at least 1.
> - For a variable x of array type: unsafe.Alignof(x) is the same as the alignment of a variable of the array's element type.

The takeaway here is, for a struct, the alignment guarantee is equal to that of largest sized field.


Now, with this knowledge under the sleeve, let's revisit our program earlier and see why the structs occupied more memory. Let's look at DemoStruct1.
``` go
type DemoStruct1 struct {
	a int8
	b int64
	c int32
}
```
Sizes (in bytes) of fields 'a', 'b' and 'c' are 1, 8 and 4 respectively. That means, 'b' is largest field in the struct. So, compiler will make whole struct 8 byte aligned. Now suppose, it starts storing the struct contents from address 64 (note, this is 8 byte aligned). 'a' will occupy one byte. i.e. byte at address 64 will be occupied by 'a'. Next, if compiler now starts placing contents of 'b' from 65, it won't be 8 byte aligned. So, compiler needs to start storing contents of 'b' from address 72 till 79, making it 8 byte aligned. So, bytes from 65-71 are padded bytes. Next, 'c' takes 4 bytes to store, taking byte numbers from 80 till 83. However, the compiler is still not done storing whole struct. Remember, it needs to make whole struct 8 byte aligned. Let's say we need to create another struct of type DemoStruct1, the compiler can't start storing it from 84 since it's not 8 byte aligned. The next struct has to be written from 88 to make it 8 byte aligned. So, in previous struct the compiler pads bytes from 84-87.
So, 'a' takes 1 + 7 padded bytes; b takes 8 bytes and c takes 4 + additional 4 padded bytes, resulting in total 24 bytes occupied by the struct

On similar lines, memory distribution of 'Demostruct2' can be explained.
``` go
type DemoStruct2 struct {
	a int8
	b int32
	c float64
}
```
Again, largest field here takes 8 bytes so whole struct needs to be 8 byte aligned. Suppose compiler starts storing from byte number 64. 'a' needs 1 byte to store. However, next field 'b' needs to 4 byte aligned, so compiler pads 3 bytes to 'a'.
This makes 'a' take 4 bytes from 64-67. 'b' takes 4 bytes from 68-71. In this case, compiler can start storing 'c' immediately after 'b' (72-79 in this case) because the number of bytes taken by 'a' and 'b' combined are 8, making next address automatically 8 byte aligned. So far, 'a', 'b' and 'c' combined take 16 bytes to store. Since next available address (80 in this case) is already 8 byte aligned, there is no need of padding at the end. So the result is, DemoStruct2 takes 16 bytes to store.

#### Summary

So in summary, memory alignment is done to prevent extra memory calls when dealing with data stored on memory.
It's done by padding some extra bytes to the data so that addresses align to a specific value.
How it's done largely depends on the hardware architecture and compiler. Should you care/worry about it? Only if you need to squeeze out that last ounce of performance for you application.
