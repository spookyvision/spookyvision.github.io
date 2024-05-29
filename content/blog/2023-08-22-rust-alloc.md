+++
title = "Custom allocators in Rust"
+++

In this article I'll explore several ways of customizing memory allocation in Rust.
If you want to follow along, you'll need [nightly](https://rust-lang.github.io/rustup/concepts/channels.html#working-with-nightly-rust) for some parts; also we're going to gloss over a lot of details first for brevity's sake – don't expect to be able to compile the introductory examples.

## Peeling back the curtain - what even is an allocation?

At its very core, allocating memory means "give me a pointer that I can use for reading and writing data".
In most systems programming, there's usually two different kinds of memory: stack and heap.
Stack allocations happen automatically as part of a function call. They are fast and cheap, but Rust needs to know your objects' required size at compile time.

```rust
fn do_something() {
    // put some numbers on the stack
    let mut items = [0u8; 200];
    // we can even change them
    items[0] = 10;
}
```

When this function is called, somewhere deep down in your computer a stack pointer is bumped to make room for these items. Let's say we're implementing a virtual machine, then something roughly along those lines needs to happen:

```rust
// initial state
let mut stack_ptr = 0xffff_ffff;

// when `do_something()` is called, reserve space for our array. 
// stacks usually grow downwards.
stack_ptr -= 200 * size_of::<u8>();

// initialize array
for idx in 0..200 {
    let item_ptr = stack_ptr + idx * size_of::<u8>();
    unsafe {
        *(item_ptr as *mut u8) = 0;
    }
}

// execute `items[0] = 10`;
let first_item_ptr = stack_ptr;
unsafe {
    *(first_item_ptr as *mut u8) = 10;
}

// return from function call
stack_ptr += 200 * size_of::<u8>();
```

Allocating *and deallocating* is blazingly fast with just one pointer addition/subtraction. Neat! 

But a dynamically sized array won't work:

```rust
fn dyn_alloc_stack(num_items: usize) {
    // error[E0435]: attempt to use a non-constant value in a constant
    let items = [0u8; num_items];
}
```

Sadface. Not to worry, there's an easy fix - *just use a `Vec`*:

```rust
fn dyn_alloc_heap(num_items: usize) {
    let mut items = vec![0u8; num_items];
    items[0] = 10;
    // we can even append stuff!
    // commented out to keep the upcoming pseudocode short.
    // items.push(123);
}
```

The function name already gave it away - the backing memory for this `Vec` comes from the other magical place: The Heap. Heap allocations are handled *at runtime* by an allocator - typically this is deferred to your operating system. What's not to like? Well… let's revisit out mock virtual machine. 
The stack pointer handling happens here as well, but we'll ignore it to focus on the heap.

```rust
// when `dyn_alloc_heap` is called, a `Vec` is constructed. 
// Internally it does something roughly like this:
let items_ptr = malloc(num_items * size_of::<u8>());
unsafe {
    *(items_ptr as *mut u8) = 10;
}

// Note that the `Vec` object itself is stored *on the stack* - only its contents live on the heap.
// Once this `Vec` goes out of scope it is dropped, which leads to:
free(items_ptr);
```

so instead of incrementing/decrementing the stack pointer we're doing two entire function calls to handle heap memory - `malloc` and `free`. What's worse, their actual implementation of those isn't even part of our program (at this point - more on that later), instead we need to ask the operating system by doing a [syscall](https://en.wikipedia.org/wiki/System_call).

To put it mildly, this is effin' expensive in comparison. Let's be clear though: this usually isn't a problem, don't optimize before you've benchmarked, yadda yadda.

But what if it *does* matter? Or you don't even *have* an operating system? Maybe because you're writing one yourself, or are working with _bare metal_ embedded devices?

## Detour: heapless
middle ground blub

## Detour: reference types
pass this vec - look at File::write or something

- separate article, dyn, box, refcell etc