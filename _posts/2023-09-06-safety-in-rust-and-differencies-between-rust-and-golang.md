---
layout: post
title: Safety in Rust and differences between Rust and Golang
date: 2022-09-07
categories: blog
tags: [Rust]
description: 
---


As software engineers, we are well aware that memory management is a significant concern. In languages like C/C++, we are given complete control over memory, which, while powerful, can make programming challenging.

Languages like Java and Go offer garbage collection (GC) mechanisms, automatically collecting memory fragments not referenced by any variables. However, GC comes at the cost of substantial resource usage.

Rust introduces a novel concept: ownership. In Rust, every memory allocation is tied to a variable. When all variables pointing to a memory fragment go out of scope or are no longer accessible, Rust reclaims that memory.

If you're familiar with Golang, I'll provide an example to illustrate the differences in memory management between Go and Rust.

Consider the following scenario:
![pic](https://whyy7777.github.io/img/ownership1.png)
In this Rust code, you might expect to push a new element to the end of the vector v. However, the compiler will raise an error:
![pic](https://whyy7777.github.io/img/ownership2.png)
This might be perplexing at first. The issue lies in how Rust manages vectors. A vector is a reference type; v is stored on the stack and contains three members: pointer (pointing to data in the heap), length (the vector's length), and capacity (the allocated memory size).

When you push an element into a vector whose length equals its capacity, Rust reallocates memory, changing the address of v[0]. If you attempt to access the original address after this reallocation, you'll encounter a null pointer error.

Now, let's look at the Go equivalent:
![pic](https://whyy7777.github.io/img/ownership3.png)
The result of this program is:
![pic](https://whyy7777.github.io/img/ownership4.png)
In Go, we use slices, which are similar to Rust's vectors. When a slice reaches its capacity, Go allocates a new memory fragment with more space. However, the address of sliceInt[0] remains unchanged during this process. Runtime garbage collection in Go doesn't immediately reclaim the space of the original fragment, allowing you to access pointToa0 without errors.

However, if you attempt to access pointToa0 after the Go Garbage Collector collects the original fragment, it will result in a null pointer error and trigger a panic.