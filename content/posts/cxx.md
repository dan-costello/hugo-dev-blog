---
date: '2025-04-27T11:40:03-04:00'
draft: true
title: 'Solving the Rust-C++ Integration Challenge: A Practical Guide'
---

Tauri is a popular Rust crate that allows developers to create desktop and mobile solutions - it appealed to me while learning Rust because I could use Rust for the backend functionality, and lean on familiar React code for the frontend UI.  While developing a recent application, a user requested a function that our development team had already implemented in C++ for other applications, but we didn't have in our Tauri UIs. Rather than rewriting the functionality in Rust, I chose to link to the existing C++ code directly to Rust to save time and maintain consistency with existing applications. This post documents my journey with this linking project as a Rust novice.

## Approaches

### Don't Reinvent the Wheel
To begin, I reviewed our existing slate of Tauri apps.  There was one other application which does call C++ code using the `libc` crate. While this looked promising for me at first, I soond discovered `libc` wouldn't work for me as it is limited to sending only basic `C` data types (ints, floats, chars) between C+ and Rust, and my use case called for involves sending vectors and strings between the two languages.  Time to continue looking...

### Finding cxx
Googling around, I found the `cxx` crate which promised safe linking of C++ and Rust code, including support for vectors and strings. Great! I started by mapping directly from my Rust source code to the function in the C++ lib. This led to numerous of errors, and it wasn't explicitly clear what I was doing wrong. I pounded the error messages through google, stack overflow, and Claude to try to find a quick fix, with no good results. Cxx still seemed like the right tool for the job, but I needed to step back a bit.

### Start with minimal reproduction
When working with new libraries, I learn best by looking at others' code and adapting it to my needs. Over time this allows me to built familiarity with a language or library and I can use them on my own projects. The maintainers of CXX crate are nice enough to include a tutorial section. To confirm everything worked on my machine, I first downloaded the tutorial repo and ran the code as-is to ensure my machine was set up correctly.  Next, I deleted this downloaded repository and started over. This time, I read through the tutorial and coded along line by line, to better understand how the pieces fit together. This worked well and allowed me to understand the methodology underlying the Cxx crate.  Once I'd completed the tutorial, I aimed to extend the tutorial with dummy data to fit my use case - calling a function with two parameters - (1) a Vector of Strings, and (2) a string.  I kept this extension super-simple to ensure I was calling functions correctly.  

#### Problem 1 - can't push to a CxxVector from Rust
In my ideal use case, I build a CxxVector/<CxxString/> (a list containing strings) in Rust and pass that as an argument to the C++ function. The problem I ran into is that Cxx doesn't allow pushing new members to a CxxVector from Rust.  My solution: Create a utility C++ function to push to a CxxString to CxxVector.

```
void push_string(std::vector<std::string> &vec, const std::string &str)
{
    vec.push_back(std::move(str));
}
```

Now I can call the `push_string` C++ function from Rust with the vector and string arguments
```
use cxx::{CxxString, CxxVector, let_cxx_string};
use ffi::push_string;

#[cxx::bridge]
mod ffi {
    unsafe extern "C++" {
        include!("cxx-demo/include/helper.h");
        fn push_string(vec: Pin<&mut CxxVector<CxxString>>, str: &CxxString);
    }
}

fn main() {
    let mut vec = CxxVector::<CxxString>::new();
    let_cxx_string!(hello = "Hello");
    let_cxx_string!(world= "World");
    push_string(vec.pin_mut(), &hello);
    push_string(vec.pin_mut(), &world);

    println!("Result is: {:?}", vec);
}
```

A working version of this example is available [here](https://codesandbox.io/p/devbox/unruffled-browser-h6qmhh) under the `src` folder.

#### Problem 2 - can't return a CxxString from C++ to Rust by Value
After solving the "vector push" issue, I encountered another challenge with returning strings from C++ to Rust. The `cxx` crate does not allow the C++ function to return a string to Rust by value - rather, it needs to send a reference/pointer (explanation [here](https://news.ycombinator.com/item?id=26565444#:~:text=std%3A%3Astring%20and%20std%3A%3Avector%20are%20not%20things%20that%20can%20exist%20by%20value%20in%20Rust)). Therefore, I can't call the original/target C++ function directly from Rust - I'll need to create another C++ wrapper function that returns a pointer to the String.

One important note: as my target C++ function findItemInList is defined in the C++ libraries that I link to, I do not need to provide the function definition in my helper.cc function (where I have defined `push_string` and `findItemFromOptions` wrapper function).  However, I do need to provide a function signature in `include/helper.h` to define its inputs and outputs.

```
include/helper.h

std::unique_ptr<std::string> findItemFromOptions(
    const std::vector<std::string> &attrs,
    const std::string &kind)
{
    std::string result = std::string(findItemInList(attrs, kind));
    return std::make_unique<std::string>(result);
}

```

### Linking to existing code
So now I have my wrapper functions and everything works in my extended version of the `cxx` tutorial and it is time to take these learnings over to my actual project.  I replicated the helper functions for pushing to a CxxVector in my feature branch and verified they ran.  Next up, I tested the wrapper function.  This did not work straight away, and failed with the ominous error that `Rust cannot catch foreign exceptions`. It took me a while to figure it out, but eventually realized that the C++ code was expecting an environment variable to be set that I didn't have in my testing environment.  Once this variable was set, the code worked!


## Building

To allow the rust compiler to link to the compiled C++ libraries, I added a build.rs to my Rust project's base folder. First, need to tell the compiler where to look for libraries:

```
println!("cargo:rustc-link-search=/path/to/libs");
```

Then link to specific library names:
```
println!("cargo:rustc-link-lib=static=lib1");
println!("cargo:rustc-link-lib=static=lib2");
println!("cargo:rustc-link-lib=lib3");
```

As we compile our software for both Linux and Windows, I needed to use some if/else blocks (i.e `if std::env::consts::OS == "linux"`) to specify different link search paths for each operating system.

## Room for Improvement
### Providing C++ signatures in header file

In an earlier implementation, I used include! macro in main.rs to include the header file that contains the signature for findItemInList.  This step appeared to be necessary so that the compiler can confirm I am calling it correctly from my wrapper function.  However, this led to issues with version control and building on other machines as I had to allow for users having the C++ repository in different locations. I was able to resolve this by removing the include! macro usage for linking to the original C++ source code, and providing a copy of the findItemInList signature in helper.h.  Although this copy/paste breaks the DRY (don't repeat yourself) principle, it cleaned up build pipeline for me. 

### Do I actually need push_string function defined?

During this work, it continued to seem unnecessary to create the push_string helper function.  If I were extending this work to work with vectors of other types, I'd have to create other helper functions to push various data types to CxxVector.  I shared the code sandbox link above with cxx maintainers to ask if I was using best practices, and will follow up here if I hear back.

### Sharing with others
I am writing this up for a couple of reasons. The first is selfish, in that it will allow me to come back and reference my thought process when linking other pieces of C++ code to Rust (and vice versa). The second is a bit more community oriented, in that there aren't many detailed examples of linking Rust to C++ code. The cxx tutorial was a great resource, and my work extends those examples for a particular use case.

## Conclusions
Being able to link Rust code to existing C++ code is a huge boon to productivity for my company (and many others, I'm guessing). Such links allow us to leverage existing code functionality without rewriting in other languages and ensuring behavior matches when called from various modules.  Big thanks to the maintainers of [cxx crate](https://github.com/dtolnay/cxx), especially its creator [dtolnay](https://github.com/dtolnay)!