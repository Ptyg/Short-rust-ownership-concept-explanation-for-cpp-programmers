<a id="readme-top"></a>

# Short Rust ownership concept explanation for C++ programmers

<details>
    <summary>Table of Contents</summary>
    <ol>
        <li>
            <a href="#start-with-example-error">Start with example error</a>
        </li>
        <li>
            <a href="#explanation">Explanation</a>
        </li>
         <li>
            <a href="#conclusion">Conclusion</a>
        </li>
    </ol>
</details>

## Start with example error

Ok, we have functionally+logically identical Rust and C++ code snippets

```rust
fn foo(nums: Vec<i32>) {

    // print vector in foo
    print!("\n[foo]: ");
    for i in &nums {
        print!("{},", i);
    }
}

fn main() {
    let nums: Vec<i32> = vec![1,2,3];
    
    // print vector
    print!("\n[main]: ");
    for i in &nums {
        print!("{},", i);
    }
    
    foo(nums);
    
    // print vector again
    print!("\n[main]: ");
    for i in &nums {
        println!("{},", i);
    }
}
```

```c++
#include <iostream>
#include <vector>

using std::vector;
using std::cout;

void foo(vector<int32_t> nums) {

    // print vector in foo
    cout << "\n[foo]: ";
    for (int32_t& i : nums){
        cout << i << ',';
    }
}

int main() {
    vector<int32_t> nums{1,2,3};

    // print vector
    cout << "\n[main]: ";
    for (int32_t& i : nums){
        cout << i << ',';
    }

    foo(nums);

    // print vector again
    cout << "\n[main]: ";
    for (int32_t& i : nums){
        cout << i << ',';
    }
    return 0;
}
```

So, as C++ developer, we assume that in both cases data will be copied because function `foo` has no reference. In other words - pass by value.

Try to compile C++ code - no problemo.

```sh
> clang++ -std=c++20 test.cpp 
> ./a.exe

[main]: 1,2,3,
[foo]: 1,2,3,
[main]: 1,2,3,
```

__But__, when we try to compile the Rust code we get an error.

```sh
> rustc test.rs

error[E0382]: borrow of moved value: `data`
  --> test.rs:23:14
   |
11 |     let data: Vec<i32> = vec![1,2,3];
   |         ---- move occurs because `data` has type `Vec<i32>`, which does not implement the `Copy` trait
...
19 |     foo(data);
   |         ---- value moved here
...
23 |     for i in &data {
   |              ^^^^^ value borrowed here after move

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
```

Why? Data is not moved. What are you talking about?

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Explanation

> Short answer: like `use-after-move` error.

> Little longer answer: in Rust world, all data without reference will be __moved__ not copied.

As a C++ developer, we need to use different tools to make shure that out code is safe. And one of the common problems is `use-after-free` \ `use-after-move`.

Here is an example.
[Took from here](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/use-after-move.html).

```c++
#include <string>
#include <vector>
#include <memory>
#include <iostream>

using std::string;
using std::vector;
using std::cout;

int main() {
    string str = "Hello, world!";
    vector<string> messages;
    
    messages.emplace_back(std::move(str));
    cout << str;
    
    return 0;
}
```

Create a string, move it to a vector and use it. Not the best option, but can be implemented. Clang, f.e., doesn't detect this error _without compile flags_.

```sh
> clang++ -std=c++20 test.cpp
> ./a.exe

>
```

So, this situation can cause big problems or even UB and Rust compiler smart enough to catch this code _without compile flags_.

The same code as C++ example, but in Rust.

```rust
fn main() {
    let str = String::from("Hello, world!");
    let mut messages = Vec::new();

    messages.push(str); // <- str is moved - messages.push_back(std::move(str))

    println!("{}", str);
}
```

```sh
> rustc test.rs

error[E0382]: borrow of moved value: `str`
 --> test.rs:7:20
  |
2 |     let str = String::from("Hello, world!");
  |         --- move occurs because `str` has type `String`, which does not implement the `Copy` trait
...
5 |     messages.push(str);
  |                   --- value moved here
6 |
7 |     println!("{}", str);
  |                    ^^^ value borrowed here after move
  |
  = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, 
run with -Z macro-backtrace for more info)
```

As you can see, problem is detected.

Same story with functions

```c++
#include <string>
#include <vector>
#include <memory>
#include <iostream>

using std::string;
using std::vector;
using std::cout;

void foo(vector<string>&& messages) { 
    cout << "[foo]: ";
    for (auto& i : messages) {
        cout << i << ' '; 
    }
}

int main() {
    vector<string> messages{"hello", "world"};
    foo(std::move(messages));

    cout << "[main]: ";
    for (auto& i : messages) {
        cout << i << ' '; 
    }

    return 0;
}
```

And again, `use-after-move` problem.

P.s. Somehow, data is printed after move bruh

```sh
> clang++ -std=c++20 test.cpp
> ./a.exe
[foo]: hello world [main]: hello world
```

Same example, Rust
```rust
fn foo(messages: Vec<String>) {
    print!("[foo]: ");
    for i in &messages {
        print!("{} ", i);
    }
}

fn main() {
    let messages = vec![String::from("Hello"), 
                        String::from("world!")];
    foo(messages); //<- vector of string is moved - foo(std::move(messages))
    
    print!("[main]: ");
    for i in &messages {
        print!("{} ", i);
    }
}
```

```sh
error[E0382]: borrow of moved value: `messages`
  --> test.rs:13:14
   |
9  |     let messages = vec![String::from("Hello"), String::from("world!")];
   |         -------- move occurs because `messages` has type `Vec<String>`, which does not implement the `Copy` trait
10 |     foo(messages);
   |         -------- value moved here
...
13 |     for i in &messages {
   |              ^^^^^^^^^ value borrowed here after move

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Conclusion

So, the main idea of Rust world - __all data is moved if there no reference__. Thanks to compiler, we can prevent `use-after-move` / `use-after-free` problems.

If you want to copy data, just use `clone()` method directly.

In C++ compiler like `clang` or `gcc` we also can detect this problems, but we need to add compile flag.

<p align="right">(<a href="#readme-top">back to top</a>)</p>
