---
title: "Rust线程"
date: {{ date }}
categories:
    - 语言
    - Rsut
    - 并发编程
---

## 摘要

发现一本讲解Rust并发编程的书籍，感觉不错，记录一下。记录Rust创建线程以及一些相关操作记录。

## 创建线程

启动一个线程，很简单，无需多言。

```rust
use std::thread;

fn start_one_thread() {
    let handle: thread::JoinHandle<()> = thread::spawn(|| {
        println!("Hello from a thread");
    });

    handle.join().unwrap();
}
```

创建N个线程

```rust
use std::thread;

fn start_n_thread() {
    const N: usize = 10;

    let handles = (0..N)
        .map(|i| {
            thread::spawn(move || {
                println!("Hello from a thread {}", i);
            })
        })
        .collect::<Vec<_>>();

    for handle in handles {
        handle.join().unwrap();
    }
}
```

## Thread Builder

通过Builder可以对线程的初始状态进行更多的控制。修改线程名字，设置栈大小等。

```rust
use std::thread;

pub fn start_one_thread_by_builder() {
    let builder = thread::Builder::new()
        .name("foo".into())
        .stack_size(32 * 1024);

    let handler = builder
        .spawn(|| {
            println!("Hello from a thread");
        })
        .unwrap();

    handler.join().unwrap();
}
```

## 当前线程

获取当前线程的信息

```rust
use std::thread;

fn create_thread() {
    let current_thread = thread::current();
    println!(
        "current thread id: {:?}, {:?}",
        current_thread.id(),
        current_thread.name()
    );

    let builder = thread::Builder::new()
        .name("foo".into())
        .stack_size(32 * 1024);
    let handler = builder
        .spawn(|| {
            let current_thread = thread::current();
            println!(
                "current thread id: {:?}, {:?}",
                current_thread.id(),
                current_thread.name()
            );
        })
        .unwrap();

    handler.join().unwrap();
}
```

通过park, unpark来控制线程的运行

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let parked_thread = thread::spawn(|| {
        println!("Parking thread");
        thread::park();
        println!("Thread unparked");
    });

    thread::sleep(Duration::from_millis(10));

    println!("Unparking thread");
    parked_thread.thread().unpark();

    parked_thread.join().unwrap();
}
```

## 并发数和当前线程数

获取当前计算机配置的最大并发数

```rust
use std::thread;

let count = thread::available_parallelism().unwrap().get();
```

affinity获取当前CPU核数

```rust
use affinity;

let cores: Vec<usize> = (0..affinity::get_core_num()).step_by(2).collect();
```

num_cpus获取当前计算机配置的CPU核数

```rust
let count = num_cpus::get();

let num = num_cpus::get_physical();
```

## sleep和park

sleep和park的区别, sleep会阻塞当前线程，park不会阻塞当前线程，park会阻塞当前线程，直到其他线程调用unpark唤醒当前线程。

使用park, unpark需要小心，避免多次调用导致线程一直阻塞。

## scoped thread

先看一下代码，来理解为什么需要scoped thread。

```rust
use std::thread;

fn main() {
    let mut a = vec![1, 2, 3];
    let mut x = 0;

    thread::spawn(move || {
        println!("hello from the first scoped thread");
        dbg!(&a);
    });

    thread::spawn(move || {
        println!("hello from the second scoped thread");
        x += a[0] + a[2];
    });
    println!("hello from the main thread");

    a.push(4);
}
```

这段代码会编译失败，因为线程外的a无法move到两个线程中。尽管两个线程对a都是只读的。要让现成外地变量同时被多个现成访问，就需要用到scope。

```rust
use std::thread;

fn main() {
    let mut a = vec![1, 2, 3];
    let mut x = 0;

    thread::scope(|s| {
        s.spawn(|| {
            println!("hello from the first scoped thread");
            dbg!(&a);
        });

        s.spawn(|| {
            println!("hello from the second scoped thread");
            x += a[0] + a[2];
        });

        println!("hello from the main thread");
    });

    a.push(4);
}
```

## ThreadLocal

Rust程序thread-local storage的实现。可以在存储数据到全局变量中，每个线程都有这个存储变量的副本。线程不会共享这个数据，副本是线程独有的，所以访问不需要同步控制。


```rust
use std::{cell::RefCell, thread};

fn main() {
    thread_local!(static COUNTER: RefCell<u32> = RefCell::new(1));

    COUNTER.with(|counter| {
        *counter.borrow_mut() += 1;
    });

    let handle1 = thread::spawn(move || {
        COUNTER.with(|counter| {
            *counter.borrow_mut() = 3;
        });
    });
    let handle2 = thread::spawn(move || {
        COUNTER.with(|counter| {
            *counter.borrow_mut() = 4;
        });
    });

    handle1.join().unwrap();
    handle2.join().unwrap();

    COUNTER.with(|counter| {
        println!("c={:?}", counter);
    });
}
```

## Move

move的作用是，线程是否需要获取变量的所有权。多个线程move int类型没有报错，因为int类型实现了copy trait。move的时候会复制值到线程中。


## 控制新建的线程

通过make_pair实现线程之间的通信。通知线程是否继续执行。

```rust
use std::thread;
use std::time::Duration;
use thread_control::*;

fn main() {
    let (flag, control) = make_pair();
    let handle = thread::spawn(move || {
        while flag.alive() {
            println!("hello world");
        }
    });
    thread::sleep(Duration::from_secs(1));
    assert_eq!(control.is_done(), false);
    control.stop();
    handle.join().unwrap();
    assert_eq!(control.is_interrupted(), false);
    assert_eq!(control.is_done(), true);
}
```

## 设置线程优先级

```rust
use std::{thread, time::Duration};

use thread_priority::{set_current_thread_priority, ThreadPriority};

fn main() {
    let handle1 = thread::spawn(|| {
        assert!(
            set_current_thread_priority(ThreadPriority::Crossplatform(0.try_into().unwrap()))
                .is_ok()
        );
        thread::sleep(Duration::from_secs(1));
        println!("hello world");
    });

    let handle2 = thread::spawn(|| {
        assert!(
            set_current_thread_priority(ThreadPriority::Crossplatform(1.try_into().unwrap()))
                .is_ok()
        );
        thread::sleep(Duration::from_secs(1));
        println!("hello rust");
    });

    handle1.join().unwrap();
    handle2.join().unwrap();
}
```

## 设置affinity

```rust
use affinity;

#[cfg(not(target_os = "macos"))]
fn use_affinity() {
    let cores: Vec<usize> = (0..affinity::get_core_num()).step_by(2).collect();
    println!("{:?}", cores);
    affinity::set_thread_affinity(&cores).unwrap();
}
```

## Panic

捕获子线程中的panic。

```rust
use std::{thread, time::Duration};

fn main() {
    println!("Hello, world!");

    let h = thread::spawn(|| {
        thread::sleep(Duration::from_secs(1));
        let result = std::panic::catch_unwind(|| {
            panic!("boom");
        });
        println!("panic caught: {}", result.is_err());
    });

    let r = h.join();
    match r {
        Ok(r) => println!("thread returned {:?}", r),
        Err(e) => println!("thread panicked with {:?}", e),
    }
}
```

通过scope生成的scope thread，任何一个线程panic，如果未被捕获，则整个scope线程都会panic。

## crossbeam scoped thread

```rust
fn main() {
    let mut a = vec![1, 2, 3];
    let mut x = 0;

    crossbeam::scope(|s| {
        s.spawn(|_| {
            println!("hello from thread");
            dbg!(&a);
        });
        s.spawn(|_| {
            println!("hello from thread");
            x += a[0] + a[2];
        });
        println!("hello from main");
    })
    .unwrap();

    a.push(4);
    assert_eq!(x, a.len());
}
```

子线程在spawn的时候，传递了一个scope，可以继续在子线程中创建孙线程。


## 总结

Rust线程的创建牵扯到了如此多的操作。充分理解概念以及使用场景，才能更好的使用。