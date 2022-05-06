---
layout: post
title: Over-Engineering A Fairly Simple Coding Challenge
categories: [algorithms, rust, unsafe]
---

I'm looking for jobs right now and I came across a seemingly blase coding challenge. It goes like this:

```
Using your favorite language, find the kth largest element in an unsorted array.
Note that it is the kth largest element in the sorted order, not the kth distinct element.
What is a second way you might do the same problem? Which approach would you prefer and why? (Optional)
```

Right off the bat there's a trivial solution: we could simply sort and dedup[^1] the array, then return the Nth largest result. If we cheat a little and use Vec, that looks like this:

[^1]: Dear Reader: You may have noticed that I misread the question, and that the algorithm doesn't need to dedup. Well, it makes for more interesting reading, so I'm doing it anyways.

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
/// The first attempt. Doesn't dedup and is pretty inefficient.
pub fn split_impl<T: PartialOrd>(slice: &[T], n: usize) -> Option<&T> {
    // This function is naturally-indexed. There is no 0th max.
    if n == 0 { return None; }

    let mut max: Option<(&T, usize)> = None;
    for (i, t) in slice.iter().enumerate() {
        max = if let Some((max_val, max_index)) = max {
            if max_val > t {Some((max_val, max_index))}
            else {Some((t, i))}
        }
        else {
            Some((t, i))
        };
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

In order to re-implement the same algorithm, I need[^2] to use `*const T` instead of `&T`. This way I don't have to contiuously allocate memory for my arena, I don't have to worry about lifetimes because pointers are owned values, and I can use nullpointers to dedupliate in-place.

[^2]: Ok, I don't *need* to use `*const T`. In fact, later, I implemented this with `&T`. Chalk this up to me thinking about this like it's C++.

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

## Benchmarks and Conclusion

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

## Update - Cleanup and Convert to References

In response to some comments, I rewrote some portions of the code.

First, I encapsulated the memory allocation within a type.

```rust
/// Wrapper for an &mut [*const T].
/// Takes care of allocation automatically.
// omitted: impl Deref, DerefMut, enum MutSliceError
#[derive(Debug)]
pub struct MutSlice<'a, T> {
    layout: Layout,
    slice: &'a mut [*const T],
}
impl<'a, T> MutSlice<'a, T> {
    pub fn try_from(slice: &'a [T]) -> Result<Self, MutSliceError> {
        let len = slice.len();
        let size = size_of::<*const T>();
        if len == 0 || size == 0 {return Err(MutSliceError::ZeroSize);}
        
        let layout = Layout::from_size_align(len * size, size).unwrap();
        let mut_slice = unsafe {
            let data = alloc_zeroed(layout) as *mut *const T;
            for i in 0..len {
                let t = slice.get(i).unwrap();
                let ptr = data.add(i);
                *ptr = t as *const T;
            }
            std::slice::from_raw_parts_mut(data, len)
        };

        Ok(Self {
            layout,
            slice: mut_slice,
        })
    }
}
impl<'a, T> std::ops::Drop for MutSlice<'a, T> {
    fn drop(&mut self) {
        unsafe {
            dealloc(self.slice.as_mut_ptr() as _, self.layout);
        }
    }
}
```

This could definitely be improved. First of all, it should really be called `FixedSizedNullableSliceRef` or something. We could get rid of the FixedSize restriction on this by implementing a `reserve` function, similar to how Vec works. I'm not seting out to fully implement Vec, but it's interesting to tinker with the mechanics behind it!

You may notice that I switched from `dealloc(self.slice.as_ptr() as _ ...` to `dealloc(self.slice.as_mut_ptr() as _ ...`. This makes sense, we're clearing the data so we have to provide it a mutable pointer. Miri didn't catch this though, and I think that's because we can freely convert between `*const T` and `*mut T`. Dealloc takes a `*mut u8` as its first parameter, so when I alias the type it converts it just fine. Since we're in an unsafe block, we can dereference the pointer as mut regardless of if the original conversion was const or mut.

This new struct works pretty well, and partially cleans up the original function. We can just create the MutSlice and then perform our algorithm:

```rust
pub fn raw_impl<T: PartialOrd>(slice: &[T], n: usize) -> Option<&T> {
    // &[T] -> &mut [*const T]
    // safety: size_of_val(layout) > 0 guaranteed by early out
    let mut ref_slice = MutSlice::try_from(slice).ok()?;

    // do it
    let mut res = None;
    unsafe {
        //...
    }
    res
} // memory deallocated on drop
```

But, looking back, I realized I really didn't need to be converting to a slice of pointers. Instead of nullifying the pointers in the MutSlice, I could just encapsulate them in Options. So I rewrote MutSlice to look like this:

```rust
struct MutSlice<'a, T> {
    layout: Layout,
    slice: &'a mut [Option<&'a T>],
}
```

This way we can have a "null reference" that is just represented by a `None`. So I went to work implementing that:

```rust
/// Re-implementation with a new MutSlice type that works with references instead of pointers.
pub fn ref_impl<T: PartialOrd>(slice: &[T], n: usize) -> Option<&T> {
    // &[T] -> &mut [&T]
    let mut ref_slice = MutSlice::try_from(slice).ok()?;

    // do it
    let mut res = None;
    for _ in 0..n {
        // Get max
        let mut max: Option<&mut Option<&T>> = None;
        for opt in ref_slice.iter_mut() {
            // these .as_deref() just convert &mut &T to &T.
            if let Some(t) = opt.as_deref() {
                if let Some(max_opt) = max.as_deref() {
                    if let Some(max_val) = max_opt {
                        if t > max_val {
                            max = Some(opt);
                        }
                        // dedup
                        // is there a different way to check if opt and max_opt point to the same memory location?
                        else if opt as *const _ != max_opt as *const _ && t == *max_val {
                            *opt = None;
                        }
                    }
                }
                else {
                    max = Some(opt);
                }
            }
        }

        // Pop max from slice
        if let Some(max_opt) = max {
            if let Some(max_val) = max_opt {
                res = Some(*max_val);
                *max_opt = None;
            }
        }
        else {
            res = None;
            break;
        }
    }
    res
}
```

I wasn't *entirely* able to avoid using pointers here. As far as I know, since & refrences automatically dereference themselves during comparison, we can't comapre the memory locations directly. Instead we need to convert them to pointers and then compare them. This operation is entirely safe because we're not derefencing them.

I then ran the same benchmarks and - somewhat to my surprise - the ref implementation outperformed the raw one, though not by much.

```
benching slice of size 16384
Vec res: None in 13.6472ms
Raw res: None in 262.9811ms
Ref res: None in 236.3619ms
benching slice of size 32768
Vec res: None in 28.3626ms
Raw res: None in 454.9425ms
Ref res: None in 450.3518ms
benching slice of size 65536
Vec res: None in 59.1616ms
Raw res: None in 884.0398ms
Ref res: None in 829.6815ms
```

I imagine that this speedup is partially caused by compiler optimizations. It makes sense that the compiler could guarantee more things about references than pointers, and so speed up its execution time. We're also not converting several pointers to references every loop, which I imagine saves a few calls.

I was also asked to make the benchmarks public, so I went ahead and [uploaded all the source code in this post to GitHub](https://github.com/ada-x64/writeups/tree/main/2022-05-05-over-engineering).