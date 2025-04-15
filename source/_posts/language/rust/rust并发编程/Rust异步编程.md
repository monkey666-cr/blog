---
title: "Rust异步编程"
date: {{ date }}
categories:
    - 语言
    - Rsut
    - 并发编程
---

## 摘要

异步编程时一种并发模型，通过在任务执行期间不阻塞线程的方式，提高系统的并发能力和相应。相比传统的同步编程，异步编程可以更好的处理I/O密集型任务和并发请求，提高系统的吞吐量和响应速度。

## Tokio

异步任务创建

```rust
#[tokio::main]
async fn main() {
    tokio::spawn(async {
        println!("hello");
    });
}
```

显式的创建tokio运行时

```rust
pub fn tokio_async() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("hello from tokio");
        rt.spawn(async {
            println!("Hello from a tokio task!");
            println!("in spawn")
        })
        .await
        .unwrap();
    });

    rt.spawn_blocking(|| println!("in spawn_blocking"));
}

fn main() {
    tokio_async();
}
```

## futures

futures是Rust异步编程的基础抽想库，为编写异步代码提供了核心的trait和类型。

- Feature trait: 表示一个异步计算的抽象，可以`.await`获取其结果。
- Stream trait: 表示一个异步的数据流，可以通过`.await`迭代获取其中的元素。
- Sink trait: 代表一个可以异步接受数据的目标。
- Executor trait: 执行futures的运行时环境。
- Utilities: 一些组合、创建futures的函数。

```rust
use futures::channel::mpsc;
use futures::executor::{self, ThreadPool};
use futures::StreamExt;

pub fn futures_async() {
    let pool = ThreadPool::new().unwrap();
    let (tx, rx) = mpsc::unbounded::<i32>();

    let fut_values = async {
        let fut_tx_result = async move {
            (0..100).for_each(|v| {
                tx.unbounded_send(v).expect("Failed to send");
            });
        };
        pool.spawn_ok(fut_tx_result);

        let fut_values = rx.map(|x| x * 2).collect();

        fut_values.await
    };

    let values: Vec<i32> = executor::block_on(fut_values);

    println!("Values={:?}, ", values);
}

fn main() {
    futures_async();
}
```

## try_join、join、select和zip

`select!`宏可以用于同时等待多个异步任务的完成，并返回最先完成的任务的结果。

```rust
use futures::future::{select, FutureExt};

let future1 = async {};
let future2 = async {};

let result = select! {
    res1 = future1 => {},
    res2 = future2 => {},
}
```

`join!`宏可以用于同时等待多个异步任务的完成，并返回所有任务的结果。

```rust
use futures::future::{select, FutureExt};

let future1 = async {};
let future2 = async {};

let (res1, res2) = join!(future1, future2);
```

```rust
use std::thread;
use std::time::Duration;

use futures::pin_mut;
use futures::{future::FutureExt, select};
use tokio::runtime::Runtime;


async fn select_all() {
    let future1 = async {
        thread::sleep(Duration::from_secs(1));
        "hello"
    }
    .fuse();
    let future2 = async {
        thread::sleep(Duration::from_secs(2));
        "world"
    }
    .fuse();

    pin_mut!(future1, future2);

    let result = select! {
        result1 = future1 => {result1},
        result2 = future2 => {result2},
    };

    println!("{:?}", result);

    thread::sleep(Duration::from_secs(3));
}

fn main() {
    let runtime = Runtime::new().unwrap();
    runtime.block_on(select_all());
}
```

`try_join!`宏也可以用于同时等待多个异步任务的完成，并返回所有任务的结果。但是在任何一个future返回错误时，整个`try_join!`宏都会返回错误。

