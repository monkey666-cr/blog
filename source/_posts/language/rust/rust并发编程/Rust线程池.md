---
title: "Rust线程池"
date: {{ date }}
categories:
    - 语言
    - Rsut
    - 并发编程
---

## 摘要

线程池，顾名思义，就是将线程预先创建好，然后放到线程池中，需要的时候，直接从线程池中获取线程，执行任务。

## rayon线程池

### 创建线程池

```rust
use rayon::ThreadPoolBuilder;

fn fib(n: usize) -> usize {
    if n == 0 || n == 1 {
        return n;
    }

    let (a, b) = rayon::join(|| fib(n - 1), || fib(n - 2));

    return a + b;
}

fn rayon_pool_create() {
    let pool = ThreadPoolBuilder::new()
        .num_threads(4)
        .thread_name(|i| format!("work-{}", i))
        .build()
        .unwrap();

    let n = pool.install(|| fib(20));

    println!("{}", n);
}

fn main() {
    rayon_pool_create();
}
```

### build_scoped线程池

```rust
scoped_tls::scoped_thread_local!(static POOL_DATA: Vec<i32>);

fn rayon_thread_scoped() {
    let pool_data = vec![1, 2, 3];

    assert!(!POOL_DATA.is_set());

    rayon::ThreadPoolBuilder::new()
        .build_scoped(
            |thread| {
                let data = pool_data.clone();
                POOL_DATA.set(&data, || thread.run())
            },
            |pool| {
                pool.install(|| {
                    thread::sleep(Duration::from_secs(1));
                    POOL_DATA.with(|d| {
                        println!("{:?}", d);
                    })
                });
                pool.install(|| {
                    POOL_DATA.with(|d| {
                        println!("{:?}", d);
                    })
                });
            },
        )
        .unwrap();

    drop(pool_data);

    thread::sleep(Duration::from_secs(5));
}
```

## threadpool库

```rust
use std::{
    sync::{mpsc::channel},
    time::Duration,
};

fn threadpool_create() {
    let pool = threadpool::ThreadPool::new(4);

    let (sender, receiver) = channel();

    for i in 0..8 {
        let sender = sender.clone();
        pool.execute(move || {
            let result = i * 2;
            sender.send(result).unwrap();
        });
    }

    for _ in 0..8 {
        let result = receiver.recv().expect("接受失败");
        println!("任务结果 {}", result);
    }
}
```

## rust_pool库

```rust
fn rusty_pool_example() {
    let pool = rusty_pool::ThreadPool::default();

    for _ in 1..10 {
        pool.execute(|| {
            println!("Hello from a rusty_pool!");
        });
    }

    pool.join();
}
```

```rust
async fn some_async_fn(x: i32, y: i32) -> i32 {
    return x + y;
}

fn rusty_pool_example2() {
    let pool = rusty_pool::ThreadPool::default();

    let handle = pool.complete(async {
        let a = some_async_fn(4, 5).await;
        let b = some_async_fn(a, 5).await;
        let c = some_async_fn(b, 5).await;
        some_async_fn(c, 5).await
    });
    handle.await_complete();

    pool.join();
}
```
