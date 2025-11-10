---
layout: post
title: Windows drivers in Rust
subtitle: My first attempt
gh-repo: zaddach/windows_kernel_programming_book_2e
gh-badge: [star, fork, follow]
tags: [rust, windows, drivers]
comments: true
mathjax: true
author: Jonas Zaddach
---

I've always wanted to have a deeper look at Windows drivers, but never got around to it.
[Recently](https://techcommunity.microsoft.com/blog/windowsdriverdev/towards-rust-in-windows-drivers/4449718) there's been some buzz from Microsoft's Nate Deisinger on the current state of Rust for Windows driver development, so
I thought I could take that as a motivation and get to know drivers a bit better. Anyways I already
had Pavel Yosifovich's ["Windows Kernel Programming 2nd Edition"](https://leanpub.com/windowskernelprogrammingsecondedition)
on my reading list, so I pulled the book out and started looking at [the labs](https://github.com/zodiacon/windowskernelprogrammingbook2e). Converting those into Rust seemed a good way to get to know both kernel
drivers in general, and the Rust ecosystem for them in particular.

## Setting up projects with cargo-wdk

Compared to earlier attempts, I thoroughly appreciated how easy it is to set up a new project with `cargo-wdk` (Follow
the description in [Towards Rust for Windows Drivers](https://techcommunity.microsoft.com/blog/windowsdriverdev/towards-rust-in-windows-drivers/4449718)). You just run `cargo wdk new --wdm ./driver`, and you have an initial
project setup that compiles (almost, you still need to fill in some details in the `driver.inx`) and gives you a
skeleton to work with. No more following ten or so steps in the README to get to a working point. Great.

## Debug output with KdPrint

Going through chapter 2, a simple dummy driver that doesn't do anything except loading, unloading and some debug printing,
I noticed some things missing. For example there's no `KdPrint`, a macro defined in C++ to conditionally compile to `DbgPrint`
in a debug configuration, and to nothing in release. But hey, there's actually an implementation of Rust's `println!`
macro, and that's way greater! So let's go for that one! Well, that should work in most cases, but you still need to heed
the warning that `println!` uses `DbgPrint` internally and as such inherits the limitation that it can print only at
`IRQL <= DIRQL`. Further, `println!` uses `alloc` internally to allocate memory. We'll discuss the limitations of using
a global allocator below.

It might be better to define a macro similar to `KdPrint`, though. [mhandb](https://github.com/microsoft/windows-drivers-rs/discussions/17) proposed an implementation that I've incorporated [here](https://github.com/zaddach/windows_kernel_programming_book_2e/blob/168e03e96e544ef8b668f6b549e0a1ba62570dd5/windows-drivers-util/src/lib.rs#L7-L32).
Especially if you want to emulate `KdPrintEx` with the ability to pass a component id (I haven't implemented that yet),
a macro might be a good idea.

## Memory allocation

Rust code assumes a global, non-failing allocator. Code in the Windows kernel can typically allocate through different
allocators, and will usually use a tag to tie the allocation to a particular driver. Currently, `cargo-wdk` will set up
`WdkAllocator` as global allocator. [This](https://github.com/microsoft/windows-drivers-rs/blob/62176115a6a54d2fe4c995b168d77cf9e7ef23ee/crates/wdk-alloc/src/lib.rs#L62) will always allocate non-paged memory with the
tag `'rust'`. As discussed in the [windows-drivers-rs issues](https://github.com/microsoft/windows-drivers-rs/issues/5),
always allocating from the non-paged pool may not be a good idea in all cases. Personally, I'd love to see the idea of
scoped allocators from [ Yoshua Wuyts in 2023](https://blog.yoshuawuyts.com/nesting-allocators/), and 
[Tyler Mandry from 2021](https://tmandry.gitlab.io/blog/posts/2021-12-21-context-capabilities/) be integrated into Rust,
but there's no RFC (yet). Microsoft seems to go a different route with custom allocator functions, as seen in Nate Deisinger's post:
```rust
impl<T> LookasideList<T> {
    pub fn new(pool_type: POOL_TYPE, tag: u32) -> Result<Arc<Self>> {
        ...
    }
}
```

## Structured Exception Handling (SEH)

Rust doesn't have native support for Structured Exception Handling. With [microseh](https://crates.io/crates/microseh),
there's a crate that can intercept structured exceptions (using C code internally), but that comes with the major
drawback that Rust cannot allocate objects inside the closure (as their `::drop()` function wouldn't be called when an
exception is thrown). So it is a good idea to only call native functions, not Rust code there.
```rust
fn guarded() -> Result<(), microseh::Exception> {
    microseh::try_seh(|| unsafe {
        // Read from an unallocated memory region. (we create an aligned not-null
        // pointer to skip the checks in read_volatile that would raise a panic)
        core::ptr::read_volatile(core::mem::align_of::<i32>() as *const i32);
    })
}
```

## Tracelooging

Tracelogging is a breeze in Rust! You just need the [tracelogging](https://crates.io/crates/tracelogging) crate that
supports being compiled in kernel drivers. Then you use [define_provider!(...)](https://github.com/zaddach/windows_kernel_programming_book_2e/blob/168e03e96e544ef8b668f6b549e0a1ba62570dd5/chapter_05/booster/src/lib.rs#L30-L34),
[PROVIDER.register()](https://github.com/zaddach/windows_kernel_programming_book_2e/blob/168e03e96e544ef8b668f6b549e0a1ba62570dd5/chapter_05/booster/src/lib.rs#L44) and [write_event!(...)](https://github.com/zaddach/windows_kernel_programming_book_2e/blob/168e03e96e544ef8b668f6b549e0a1ba62570dd5/chapter_05/booster/src/lib.rs#L46-L50) like you would in C/C++.

## Overall impression

Windows drivers in Rust have come a long way. I've worked my way through the book till chapter 7, and I haven't hit a
major roadblock in my Rust conversion yet. There might be some C/C++ macros missing that I had to redefine in a helper
library, e.g., [IoGetCurrentIrpStackLocation(...)](https://github.com/zaddach/windows_kernel_programming_book_2e/blob/168e03e96e544ef8b668f6b549e0a1ba62570dd5/windows-drivers-util/src/lib.rs#L34-L53). Further, some constants are defined
as a different type than the fields they're used with (e.g., `wdk_sys::_MODE::KernelMode` is defined as a `c_int`, but is
expected to be an `i8` when used with `MmMapLockedPagesSpecifyCache(...)`). But overall, I'm impressed by the work
the team at Microsoft has put in and I'm exited for a time when Windows (production) drivers in Rust will be possible!

You can have a look at my translated examples [here](https://github.com/zaddach/windows_kernel_programming_book_2e) (work in progress). 


