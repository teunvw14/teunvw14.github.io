+++
title = "Data Structures in C: Linked List"
date = 2024-11-09
+++

In this post, you'll learn how you can 

# Data structures
A data structure is a way to structure data. Right. Makes sense. But what does that actually mean? And why do we need to structure our data? 

But why do we need data structures? 

# Why C?
Because it's so bare-bones that it doesn't obfuscate anything. Creating a data structure inherently means specifying how data is structured *in memory*. So to make data structures, we need a language that can read and write directly from and to memory. Trying to create a new data structure in Python, for example, would be impossible. Python manages memory for the programmer, which is great, because it can save programmers a lot of time. But it also means that you can't specify how Python interacts with memory. Hence why we need a low level programming language like C.

To be fair, there are some other programming languages that you could use, like C++ or Rust. But C gives me this kind of nostalgic feeling. 

# Linked List

So, what is a linked list then? Let's take a look at the image below.


# Requirements.

To follow along, you will need two things: 
- A compiler: Make sure you have `gcc` or `clang` installed so you can compile the C code we'll be writing. 
- An editor: VSCode with the C/C++ extension works great, but obviously you can use whichever editor you want. 

# Let's get started.

Start out by making a folder 'lilist'. We will be abbreviating 'linked list' in our code to 'lilist' to keep the code a little more readable. 

First, in this new folder create a file `lilist.h`. In this new file, let's define the `struct`s for the linked list.

```C
struct lilist_node {
    struct lilist_node* next;
    int value;
};
typedef struct lilist_node lilist_node;

/* A singly linked list with maximum length __INT64_MAX__. */ 
typedef struct lilist {
    int64_t length;
    lilist_node* head;
    lilist_node* tail;
} lilist;

int length() {
    return 12;
}
```
