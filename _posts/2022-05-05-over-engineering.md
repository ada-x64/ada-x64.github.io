---
layout: post
title: Over-Engineering A Simple Code Test
categories: [algorithms, unsafe, rust]
---

I'm looking for jobs right now and I came across a seemingly blase coding challenge. It goes like this:

```
Using your favorite language, find the kth largest element in an unsorted array.
Note that it is the kth largest element in the sorted order, not the kth distinct element.
What is a second way you might do the same problem? Which approach would you prefer and why? (Optional)
```

Right off the bat there's a trivial solution: we could simply sort and dedup* the array, then return the Nth largest result. If we cheat a little and use Vec, that looks like this:

```rust
fn vec_impl<T: Ord>(slice: &[T], n: usize) -> Option<&T> {
    let mut vec: Vec<&T> = slice.iter().collect();
    vec.sort();
    vec.dedup();
    let mut max = None;
    for _ in 0..n {
        max = vec.pop();
    }
    max
}
```

What a clean solution! This is essentially what we'll have to do in order to implement this anyways, but I want to see if I can do this without Rust's built-in magics - and with fewer iterations.

## Slice To Meet You

My first attempt got to here:

```rust
fn split_impl<T: PartialOrd>(slice: &[T], n: usize) -> Option<&T> {
    // This function is naturally-indexed. There is no 0th max.
    if n == 0 { return None; }

    let mut max: Option<(&T, usize)> = None;
    let mut i: usize = 0;
    for t in slice {
        max = if let Some((max_val, max_index)) = max {
            if max_val > t {Some((max_val, max_index))}
            else {Some((t, i))}
        }
        else {
            Some((t, i))
        };
        i += 1;
    }
    max.and_then(|(max_val, max_index)| {
        if n <= 1 {
            Some(max_val)
        }
        else {
            // Because slices have to be contiguously spaced, we can't just do this:
            // let slice = &[&slice[..i], &slice[i+1..]].concat();
            // nth_largest(slice, n-1)
            // If we were copying arrays around, we *could*. But we're working with slices!
            // Instead we have to split here! 
            match (
                split_impl(&slice[..max_index], n-1),
                split_impl(&slice[max_index+1..], n-1)
            ) {
                (None, None) => None,
                (None, res) => res,
                (res, None) => res,
                (Some(a), Some(b)) => Some( if a > b {a} else {b}),
            }
        }
    })
}
```

While this certainly works, it bothers me! We're running this function *way* more than we need to, >= log(n) times in fact. I'm *basically* making a really inefficient binary search tree. It would be faster just to sort the array and get the nth largest! Not to mention, this function doesn't even dedup!

In fact, if we're working with immutable slices, we *can't* dedup. Slices are a contiguous view into statically allocated memory, so we can't put gaps in our slices. Not unless we're working with a mutable slice. ... so what if we converted our `&[T]` to an `&mut [&T]`?

## In the Arena

I decided to keep the signature of my function, and to avoid using a vec if at all possible. It turns out that this is entirely possible, but a bit tedious - and *very* unsafe.

The idea is to convert the slice to a contiguous block of references. The instinct is to do this with arrays, but Rust array declarations need a `const 'static` size parameter, which we can't extract from a slice. So, we have to manually allocate the data. Here, I'm just allocating a chunk of memory the same size as the slice. This method of allocation is called linear or arena allocation, and it's one of the simplest forms of memory allocation out there.

I first heard about arena allocators while trying to implement a graph-based node editor for Sundile and I ran into the issue of implementing a self-referencing struct. [This article](https://manishearth.github.io/blog/2021/03/15/arenas-in-rust/) provides what I think is a really good introduction.

In order to re-implement the same algorithm, I need to use `*const T` instead of `&T`. This way I don't have to contiuously allocate memory for my arena, I don't have to worry about lifetimes because pointers are owned values, and I can use nullpointers to dedupliate in-place.

For some more information on how `*const T` differs from `&T` check out the [nomicon](https://doc.rust-lang.org/nomicon/subtyping.html#variance).

Anyways, here's the implementation:

```rust
use std::{mem::*, alloc::*};
pub fn raw_impl<T: PartialOrd>(slice: &[T], n: usize) -> Option<&T> {
    // Early out & avoid bad alloc
    let len = slice.len();
    let size = size_of::<*const T>(); // Pretty much guaranteed to be 8, but doesn't hurt to be sure.
    if len == 0 || size == 0 || n == 0 {return None;}
    
    // This layout initializer takes the number of bytes and the alignment.
    // The Layout struct is just this:
    // pub struct Layout {
    //     size_: usize,
    //     align_: NonZeroUsize,
    // }
    let layout = Layout::from_size_align(len * size, size).unwrap();

    // Here's the magic trick:
    // &[T] -> &mut [*const T]
    // There's no in-built way to do this that I know of - and probably for good reason.
    // Note on safety: size_of_val(layout) > 0 guaranteed by early out
    let ref_slice = unsafe {
        let data = alloc_zeroed(layout) as *mut *const T;
        for i in 0..len {
            let t = slice.get(i).unwrap(); // Panic if T is unallocated.
            let ptr = data.add(i); // Gets the pointer at the given offset.
            *ptr = t as *const T; // Assigns it!
        }
        std::slice::from_raw_parts_mut(data, len)
    };

    // Get the nth max!
    let mut res = None;
    unsafe {
        for _ in 0..n {
            // Get max
            let mut max: Option<&mut *const T> = None;
            for ptr in ref_slice.iter_mut() {
                if let Some(t) = ptr.as_ref() { // checks for null
                    if let Some(max_ptr) = &max {
                        if let Some(max_val) = max_ptr.as_ref() {
                            if t > max_val {
                                max = Some(ptr);
                            }
                            else if *ptr != **max_ptr && t == max_val { //dedup
                                *ptr = std::ptr::null();
                            }
                        }
                    }
                    else {
                        max = Some(ptr);
                    }
                }
            }

            // Pop max from slice
            if let Some(ptr) = max {
                res = ptr.clone().as_ref();
                *ptr = std::ptr::null::<T>();
            }
            else {
                res = None;
                break;
            }
        }
    }

    //dealloc
    unsafe {
        dealloc(ref_slice.as_ptr() as _, layout);
    }

    res
}
```

I originally had some trouble because I misunderstood `Layout::from_size_align`. I had it stuck in my head that the first parameter described the number of elements, similar to how `slice::from_raw_parts` works. It's bytes though, which is fairly obvious looking back, but without [Miri](https://github.com/rust-lang/miri) I wouldn't have caught it. I highly reccommend it.

# Benchmarks and Conclusion

So, overall this is pretty good! No memory leaks or heap corruption thanks to the watchful eyes of Miri. I ran some benchmarks on it as well, passing in slices of a large, randomly populated array, and a size of n = slice.len(). Here are the results:

```
benching slice of size 16384
Vec res: None in 13.322ms
Raw res: None in 208.5944ms
benching slice of size 32768
Vec res: None in 19.6034ms
Raw res: None in 321.789ms
benching slice of size 65536
Vec res: None in 35.8899ms
Raw res: None in 620.0199ms
```

So, not as efficient as the std library's implementation - but I really didn't expect it to be. This was more an exercise in unsafe Rust than anything. One thing I'm sure could help is to condense the allocation so that nullptrs are moved to the back of the arena.

Part of the slowness of this algorithim is probably due to the fact that I'm explicity avoiding allocating any more memory than the slice requires. So the algorithm has O(1) space complexity, but O(n^2) time complexity in the worst case (n = slice.len()). A pretty silly algorithm, but I like it!

Finally, a few notes on my interpretation of the problem. I realized looking back that I think I misread the question, and that dedupliaction wasn't necessary. That pretty much means my motivation for manually allocating was bunk - but again, I don't mind! I learned something, and that's the important part. 