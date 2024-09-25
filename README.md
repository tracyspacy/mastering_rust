
 
### Rust Ownership Basics : Immutable Reference, mutable Reference, Rc and Refcell

tldr

```
          Avoid `clone()` options ^_^
                |
    ---------------------------------------------
    |                                            |
Read-only access?                          Need to mutate?
    |                                            |
    --------------------                    -------------------
    |                  |                    |                 |
No Ownership?   Shared Ownership?  Exclusive mutability?   Shared mutability?
    |                  |                    |                 |
  Use &T           Use Rc<T>           Use &mut         Use Rc<RefCell<T>>
```

Rust Ownership rules:

- Each value in Rust has an *owner*.
- There can only be one owner at a time.
- When the owner goes out of scope, the value will be dropped.[1]

In this post, we will discuss heap-allocated variables and when to use references (&), mutable references (&mut), Rc, or RefCell.

Simple basics from the Book:
 
To ensure memory safety after we assign s1 to s2 (shallow copy), Rust considers `s1` as no longer valid - s1 goes out of scope , ownership is transfered to s2. Therefore, it is impossible to print s1 afterward.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;                 // ownership moved to s2
    println!("{s1} {s2}");      // s1 is no longer valid
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=14dca9f937e10df439d964c64d983857)

In this case, we might be tempted to use clone() (deep copy of heap data) to satisfy the compiler:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();        // deep copy of s1
    println!("{s1},{s2}!");     // we can print s1 and s2
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=14dca9f937e10df439d964c64d983857)

But copying heap data is an expensive operation and to achieve the same result for this read-only functionality we have nore efficient option without transfering ownership or deep copy : an immutable refference.


### Immutable refference (&T)


```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = &s1;        // borrowing s1 ie creating reference
    println!("{s1},{s2}!");     // we can print s1 and s2
}
```

[playgroung](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=4bc09ce6a93535fa7bda99d56ed926cb)

So we can use an immutable refference when we do not need to take ownership of variable and we do not need to mutate it.

We can have multiple immutable references to a variable, but these references are only valid for as long as the variable itself remains in scope. Once the variable goes out of scope, all its immutable references become invalid.

But what if different parts of our code need to access the variable simultaneously?

```rust
fn main() {
    let s2;
    {
        let s1 = String::from("hello");
        s2 = &s1;   // borrowing s1, i.e., creating a reference
    }               // s1 goes out of scope, so the borrowed value is no longer valid
    println!("{s2}"); // Error: cannot print s2 because the borrowed value does not live long enough
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=71a9c5af25405522e9300420e4e247a4)

If we will try to compile this code we will get error:`borrowed value does not live long enough`.

This is where smart pointers like Rc can help.


### Refference Count (Rc<T>) (single-threaded)


[`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html "struct std::rc::Rc") stands for Reference Count. It provides shared ownership of a value of type `T`, allocated on the heap. It allows multiple parts of a program to have ownership of the same data, and the value will only be dropped when the last reference goes out of scope.

Invoking [`clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html#tymethod.clone "method std::clone::Clone::clone") on Rc produces a new pointer to the same allocation in the heap.[2]

```rust
use std::rc::Rc;

fn main() {
    let s2;
    {
        let s1 = Rc::new(String::from("hello")); // Create a reference-counted string.
        s2 = Rc::clone(&s1); //Cloning the s1 produces a new pointer to the same allocation in the heap
    }                        // s1 goes out of scope, but the String it points to remains valid because s2 still references it
    println!("{s2}");        // we can print s2
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=e14c17e43e7be4077460eb0b474e9cd6)

Unlike an immutable reference, Rc allows multiple owners of the same value. So even though `s1` goes out of scope, the value it points to remains valid because `s2` is still referencing it. The reference count ensures the value will only be deallocated when the last owner (`s2` in this case) is dropped.

Similar to an immutable reference (`&T`), `Rc<T>` allows multiple references to data on the heap, **without allowing mutation by default**.

```rust
use std::rc::Rc;

fn main() {
    let s1 = Rc::new(String::from("hello"));
    let s2 = Rc::clone(&s1);
    let s3 = Rc::clone(&s1);
    println!("{s1} {s2} {s3}");
    // s2 and s3 both point to the same memory location as s1.
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=544f474c04a5a61490f057d89afb6dc3)

These are some primitive examples that simply help illustrate the logic. For more advanced examples and use cases, you can refer to the official documentation [std::rc - Rust](https://doc.rust-lang.org/std/rc/index.html)

As we mentioned above, immutable reference and Rc are both not allowing mutation as default, but what if we need to mutate data on the heap?


### Mutable reference (&mut T)


When we need to mutate a borrowed value, the most obvious option is to use mutable reference (& mut).

```rust
fn main() {
    let mut s1 = String::from("hello");
    change(&mut s1);
    println!("{s1}");
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=7d9ee65cbd7a9dded369c581fdc8e9dc)

Mutable references have one big restriction: if you have a mutable reference to
a value, you can have no other simultaneous references to that value.

It is not allowed to have immutable refference if there is already mutable reference exists.

```rust
fn main() {
    let mut s1 = String::from("hello");
    let s2 = &mut s1;        // mutable borrowing s1
    let s3 = &s1;            // cannot borrow `s1` as immutable because it is also borrowed as mutable
    println!("{s1},{s2},{s3}!");     
}
```

It's also not allowed to have two simultaneous mutable references to the same value:

```rust
fn main() {
    let mut s1 = String::from("hello");
    let s2 = &mut s1;        // mutable borrowing s1
    let s3 = &mut s1;       // cannot borrow `s1` as mutable more than once at a time
    println!("{s1},{s2},{s3}!");     
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=8c2f4c0d3b795a3a6bbc0620d86fff3d)

However, Rust's borrowing rules don’t allow more than one mutable reference at a time, which can be limiting in certain scenarios. For example, what if you need multiple parts of your program to mutate the same data simultaneously, without violating Rust’s safety? This is where the combination of `Rc<T>` and `RefCell<T>` comes into play, allowing shared ownership *and* mutation.


### Combining RefCell and Rc we can have multiple owners of mutable data. (single-threaded)


A common way to use `RefCell<T>` is in combination with `Rc<T>`. Recall that `Rc<T>` lets you have multiple owners of some data, but it only gives immutable access to that data by default. If you have an `Rc<T>` that holds a `RefCell<T>`, you can get a value that can have multiple owners *and* that you can mutate! [3]

```rust
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let s1 = Rc::new(RefCell::new(String::from("hello")));

    let s2 = Rc::clone(&s1);
    let s3 = Rc::clone(&s1);

    // Now both s2 and s3 can mutate the shared string
    s2.borrow_mut().push_str(", world");
    s3.borrow_mut().push_str("!");

    println!("{:?}", s1);  // hello, world!
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=f523b9c75de6c9528e672ea75a9cc276)

But of course true power of combining Rc and RefCell comes with complex data structures such as linked lists, trees etc.

Binary tree example [4]:

```rust
use std::cell::RefCell;
use std::fmt::Debug;
use std::rc::Rc;

#[derive(Debug, PartialEq, Eq, Clone)]
pub struct TreeNode<T> {
    pub val: T,
    pub left: Option<Rc<RefCell<TreeNode<T>>>>,
    pub right: Option<Rc<RefCell<TreeNode<T>>>>,
}
impl<T> TreeNode<T> {
    pub fn new(val: T) -> Self {
        TreeNode {
            val,
            left: None,
            right: None,
        }
    }
}

pub fn preorder_traversal<T: Debug>(root: &Option<Rc<RefCell<TreeNode<T>>>>) {
    match root {
        None => {}
        Some(node) => {
            let borrowed_node = node.borrow();
            println!("{:?}", borrowed_node.val);
            preorder_traversal(&borrowed_node.left.clone());
            preorder_traversal(&borrowed_node.right.clone());
        }
    }
}

pub fn inorder_traversal<T: Debug>(root: &Option<Rc<RefCell<TreeNode<T>>>>) {
    match root {
        None => {}
        Some(node) => {
            let borrowed_node = node.borrow();
            inorder_traversal(&borrowed_node.left.clone());
            println!("{:?}", borrowed_node.val);
            inorder_traversal(&borrowed_node.right.clone());
        }
    }
}

pub fn postorder_traversal<T: Debug>(root: &Option<Rc<RefCell<TreeNode<T>>>>) {
    if let Some(node) = root {
        let borrowed_node = node.borrow();
        postorder_traversal(&borrowed_node.left.clone());
        postorder_traversal(&borrowed_node.right.clone());
        println!("{:?}", borrowed_node.val);
    }
}

pub fn calculate_max_depth<T: Debug>(root: &Option<Rc<RefCell<TreeNode<T>>>>) -> i32 {
    match root {
        None => return 0,
        Some(node) => {
            let borrowed_node = node.borrow();
            let left_depth = calculate_max_depth(&borrowed_node.left.clone());
            let right_depth = calculate_max_depth(&borrowed_node.right.clone());
            if left_depth > right_depth {
                return left_depth + 1;
            } else {
                return right_depth + 1;
            }
        }
    }
}

/*
               1
            /    \
           2      3
         /  \    /  \
        4    5  6    7

*/

fn main() {
    let mut root = TreeNode::new(1);
    root.left = Some(Rc::new(RefCell::new(TreeNode::new(2))));
    root.right = Some(Rc::new(RefCell::new(TreeNode::new(3))));
    if let Some(left_node) = root.left.as_mut() {
        //same as with root we should make root.left mut to modify
        left_node.borrow_mut().left = Some(Rc::new(RefCell::new(TreeNode::new(4))));
        // borrowing is obtain mut ref to Refcall containing root.left,
        // ie to access and mutate the left field of the root inside the RefCell contained within the Rc.
    }
    if let Some(right_node) = root.left.as_mut() {
        right_node.borrow_mut().right = Some(Rc::new(RefCell::new(TreeNode::new(5))));
    }

    if let Some(left_node) = root.right.as_mut() {
        left_node.borrow_mut().left = Some(Rc::new(RefCell::new(TreeNode::new(6))));
    }
    if let Some(right_node) = root.right.as_mut() {
        right_node.borrow_mut().right = Some(Rc::new(RefCell::new(TreeNode::new(7))));
    }

    println!("{:?}", root);
    println!("Preorder Traversal:");
    preorder_traversal(&Some(Rc::new(RefCell::new(root.clone()))));
    println!("Inorder Traversal:");
    inorder_traversal(&Some(Rc::new(RefCell::new(root.clone()))));
    println!("Postorder Traversal:");
    postorder_traversal(&Some(Rc::new(RefCell::new(root.clone()))));
    println!(
        "Max depth is: {:?}",
        calculate_max_depth(&Some(Rc::new(RefCell::new(root))))
    );
}
```

[playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=6819e1f13f8f26f4c08dd5cdca27cc01)


**Summary:**

- Use &T for non-owning, read-only access.
- Use &mut T for non-owning, exclusive mutable access.
- Use Rc<T> for shared ownership when you don't need mutability.
- Combine Rc<T> with RefCell<T> for shared ownership with mutability.


Source:

[1] [What is Ownership? - The Rust Programming Language](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html)

[2][ std::rc - Rust](https://doc.rust-lang.org/std/rc/index.html)

[3][RefCell&lt;T&gt; and the Interior Mutability Pattern - The Rust Programming Language](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)

[4][algos_data_structures_rust/src/trees/binary_trees/binary_tree.rs at master · tracyspacy/algos_data_structures_rust · GitHub](https://github.com/tracyspacy/algos_data_structures_rust/blob/master/src/trees/binary_trees/binary_tree.rs)
