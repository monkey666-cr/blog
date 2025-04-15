---
title: "Rust同步原语"
date: {{ date }}
categories:
    - 语言
    - Rsut
    - 并发编程
---

## 摘要

学习一下线程共享的一些方式和库

## cow

copy-on-write的缩写，写时复制。

```rust
let s1 = String::new("hello");
let s2 = s1;  // s1和s2共享同一份内存

s2.push_str(" world");  // s2会进行写操作，会触发cow，s1也会进行写操作，s1会进行深拷贝，s2会进行浅拷贝。
```

标准库中`std::borrow::Cow`类型是一个智能指针，提供了写时克隆的功能。他可以封装并提供对借用数据的不可变访问，当需要进行修改或获取所有权时，他可以惰性的克隆数据。

```rust
use std::borrow::Cow;

fn main() {
    let origin = String::from("hello");
    let mut cow = Cow::from(&origin);
    assert_eq!(cow, "hello");

    let s = &cow;
    assert_eq!(s, "hello");
    assert_eq!(s.len(), cow.len());

    let s: &mut str = cow.to_mut();
    s.make_ascii_uppercase();
    assert_eq!(s, "HELLO");
    assert_eq!(origin, "hello");
}
```

## box

`Box<T>`提供了Rust中最简单的堆分配形式。Box为这个分配提供了所有权，并在超出作用域时释放其内容。

```rust
let val: u8 = 5;
let boxed: Box<u8> = Box::new(val);

let boxed: Box<u8> = Box::new(5);
let val: u8 = *boxed;
```

## Cell、RefCell、OnceCell、LzayCell和LazyLock

Cell和RefCell都是Rust中的线程不安全的类型，他们可以在线程不安全的环境中使用。都是可共享的可变容器。

### Cell

`Cell<T>`允许在不违反借用规则的前提下，修改其包含的值。Cell中的值不再拥有所有权，只能通过get和set方法来访问。set方法可以在不获取所有权的情况下修改值。适用于简单的单值容器，如整数或字符。

```rust
use std::cell::Cell;

fn main() {
    let x = Cell::new(452);
    let y = &x;

    x.set(10);

    println!("y: {}", y.get());
}
```

### RefCell

`RefCell<T>`提供了内部可变性，可以在运行时检查借用规则，并允许在运行时修改值。通过`borrow`和`borrow_mut`方法进行不可变和可变借用。借用必须在作用域结束前归还，否则会panic。适用于包含多个字段的容器。

```rust
use std::{cell::RefCell, ops::Deref};

fn main() {
    let x = RefCell::new(42);

    {
        let y = x.borrow();
        println!("y: {}", y);
    }

    {
        let mut z = x.borrow_mut();
        *z = 10;
    }

    println!("x: {:?}", x.borrow().deref());
}
```

### OnceCell

`OnceCell`是Rust中的单线程原子单值容器，用于在多线程环境中安全地存储单个值。它提供了`get`和`set`方法来获取和设置值。

```rust
use std::cell::OnceCell;

fn main() {
    let cell = OnceCell::new();
    assert!(cell.get().is_none());

    let value = cell.get_or_init(|| "Hello".to_string());
    assert_eq!(value, "Hello");
    assert!(cell.get().is_some());
}
```

### LazyCell、LazyLock

```rust
use std::cell::LazyCell;

fn main() {
    let lazy = LazyCell::new(|| {
        println!("initializing...");
        40
    });

    println!("ready");
    println!("{}", *lazy);
    println!("{}", *lazy);
}
```

在第一次访问的时候会初始化，但是并不是线程安全的。

```rust
use std::{collections::HashMap, sync::LazyLock};

static HASHMAP: LazyLock<HashMap<i32, String>> = LazyLock::new(|| {
    println!("initializing...");
    let mut m = HashMap::new();
    m.insert(18, "hello 18".to_string());
    m.insert(10, "hello 10".to_string());
    m
});

fn main() {
    println!("ready");
    std::thread::spawn(|| {
        println!("hashmap: {:?}", HASHMAP.get(&18));
    })
    .join()
    .unwrap();
    print!("{:?}", HASHMAP.get(&10));
}
```

## rc

Rc是使用引用计数的智能指针，用于共享所有权。Rc在处理循环引用时需要额外注意，因为循环引用会导致引用计数无法降为0，从而导致内存泄漏。解决这个问题，可以使用Weak类型。

```rust
use std::{cell::RefCell, collections::HashMap, rc::Rc};

fn rc_refcell_example() {
    let shared_map = Rc::new(RefCell::new(HashMap::new()));

    {
        let mut map = shared_map.borrow_mut();
        map.insert("africa", 92388);
        map.insert("kyoto", 11837);
        map.insert("piccadilly", 11826);
        map.insert("marbles", 38);
    }

    let total = shared_map.borrow().values().sum::<i32>();
    println!("total: {}", total);
}

fn main() {
    let data = Rc::new(100);

    let r1 = data.clone();
    let r2 = data.clone();

    println!("{}", r1);
    println!("{}", r2);

    rc_refcell_example();
}
```

Rc不是线程安全的，如果需要线程安全，可以使用Arc。
