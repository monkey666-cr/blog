---
title: "Tokio"
date: {{ date }}
categories:
    - 语言
    - Rsut
    - 并发编程
---

## 摘要

Tokio是一个基于事件驱动、非阻塞I/O的平台，用于使用Rust编写异步应用程序。在高层次上，他提供了一些主要组件：

- 用于处理异步任务的工具，包括同步原语、通道、超时、睡眠和间隔。
- 执行异步I/O操作的API，包括TCP和UDP sockets、文件操作以及进程和信号管理。
- 用于执行异步代码的运行时，包括任务调度器、基于操作系统事件队列的I/O驱动程序（epoll， kqueue， IOCP等），以及高性能计时器。

## 异步运行时

```rust
use tokio::process::Command;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let output = Command::new("echo").arg("hello").arg("world").output();
    let output = output.await?;

    println!("{:?}", output);

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");

    Ok(())
}
```

```rust
use std::process::Stdio;

use tokio::{
    io::{AsyncBufReadExt, BufReader},
    process::Command,
};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut cmd = Command::new("cat");

    cmd.stdout(Stdio::piped());

    let mut child = cmd.spawn().expect("failed to spawn command");

    let stdout = child
        .stdout
        .take()
        .expect("child did not have a handle to stdout");

    let mut reader = BufReader::new(stdout).lines();

    tokio::spawn(async move {
        let status = child
            .wait()
            .await
            .expect("child process encountered an error");
        println!("child status was: {}", status);
    });

    while let Some(line) = reader.next_line().await? {
        println!("Line: {}", line);
    }

    Ok(())
}
```

## 同步原语

### Mutex

```rust
use std::sync::Arc;

use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let data1 = Arc::new(Mutex::new(0));
    let data2 = Arc::clone(&data1);

    tokio::spawn(async move {
        let mut lock = data2.lock().await;
        *lock += 1;
    });

    {
        let mut lock = data1.lock().await;
        *lock += 1;
    }

    println!("data1: {:?}", data1.lock().await);
}
```

### RwLock

```rust
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let lock = RwLock::new(5);

    {
        let r1 = lock.read().await;
        let r2 = lock.read().await;
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    }

    {
        let mut w = lock.write().await;
        *w += 1;
        assert_eq!(*w, 6);
    }
}
```

### Barrier

```rust
use std::sync::Arc;

use tokio::sync::Barrier;

#[tokio::main]
async fn main() {
    let mut handles = Vec::with_capacity(10);
    let barrier = Arc::new(Barrier::new(10));
    for _ in 0..10 {
        let c = barrier.clone();
        handles.push(tokio::spawn(async move {
            println!("before wait");
            let wait_result = c.wait().await;
            println!("after wait");
            wait_result
        }));
    }

    let mut num_leaders = 0;
    for handle in handles {
        let wait_result = handle.await.unwrap();
        if wait_result.is_leader() {
            num_leaders += 1;
        }
    }

    assert_eq!(num_leaders, 1);
}
```

### Notify

```rust
use std::sync::Arc;

use tokio::sync::Notify;

#[tokio::main]
async fn main() {
    let notify = Arc::new(Notify::new());
    let notify2 = notify.clone();

    let handle = tokio::spawn(async move {
        notify2.notified().await;
        println!("received notification");
    });

    println!("sending notification");
    notify.notify_one();

    handle.await.unwrap();
}
```


```rust
use std::sync::Arc;

use tokio::sync::Notify;

#[tokio::main]
async fn main() {
    let notify = Arc::new(Notify::new());
    let notify2 = notify.clone();

    let notified1 = notify.notified();
    let notified2 = notify.notified();

    let _handle = tokio::spawn(async move {
        println!("sending notifications");
        notify2.notify_waiters();
    });

    notified1.await;
    notified2.await;

    println!("received notifications");
}
```

### Semaphore

```rust
use tokio::sync::{Semaphore, TryAcquireError};

#[tokio::main]
async fn main() {
    let semaphore = Semaphore::new(3);

    let _a_permit = semaphore.acquire().await.unwrap();
    let _two_permits = semaphore.acquire_many(2).await.unwrap();

    assert_eq!(semaphore.available_permits(), 0);

    let permit_attempt = semaphore.try_acquire();
    assert_eq!(permit_attempt.err(), Some(TryAcquireError::NoPermits));
}
```

### OnceCell


```rust
use tokio::sync::OnceCell;

async fn some_computation() -> u32 {
    1 + 1
}

static ONCE: OnceCell<u32> = OnceCell::const_new();

#[tokio::main]
async fn main() {
    let result = ONCE.get_or_init(|| some_computation()).await;

    assert_eq!(*result, 2);
}
```

```rust
use tokio::sync::OnceCell;

static ONCE: OnceCell<u32> = OnceCell::const_new();

async fn get_global_integer() -> &'static u32 {
    ONCE.get_or_init(|| async { 1 + 1 }).await
}

#[tokio::main]
async fn main() {
    let result = get_global_integer().await;

    assert_eq!(*result, 2);
}
```

## 通道

### mpsc

```rust
use tokio::sync::mpsc;

async fn some_computation(input: u32) -> String {
    format!("the result of computation {}", input)
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    tokio::spawn(async move {
        for i in 0..10 {
            let res = some_computation(i).await;
            tx.send(res).await.unwrap();
        }
    });

    while let Some(res) = rx.recv().await {
        println!("got = {}", res);
    }
}
```

### oneshot

单次通道支持从单个生产者向单个消费者发送单个值。通常，这种通道用于将计算的结果发送给等待者。

```rust
use tokio::sync::oneshot;

async fn some_computation() -> String {
    "represents the result of the computation".to_string()
}

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        let res = some_computation().await;
        tx.send(res).unwrap();
    });

    let res = rx.await.unwrap();

    println!("{res}");
}
```

### broadcast(mpmc)

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, mut rx1) = broadcast::channel(16);
    let mut rx2 = tx.subscribe();

    tokio::spawn(async move {
        assert_eq!(rx1.recv().await.unwrap(), 10);
        assert_eq!(rx1.recv().await.unwrap(), 20);
    });

    tokio::spawn(async move {
        assert_eq!(rx2.recv().await.unwrap(), 10);
        assert_eq!(rx2.recv().await.unwrap(), 20);
    });

    tx.send(10).unwrap();
    tx.send(20).unwrap();
}
```

### watch(spmc)

```rust
use tokio::sync::watch;
use tokio::time::{self, Duration, Instant};

use std::io;

#[derive(Debug, Clone, Eq, PartialEq)]
struct Config {
    timeout: Duration,
}

impl Config {
    async fn load_from_file() -> io::Result<Config> {
        Ok(Config {
            timeout: Duration::from_secs(3),
        })
    }
}

async fn my_async_operation() {}

#[tokio::main]
async fn main() {
    let mut config = Config::load_from_file().await.unwrap();
    let (tx, rx) = watch::channel(config.clone());

    tokio::spawn(async move {
        loop {
            time::sleep(Duration::from_secs(10)).await;
            let new_config = Config::load_from_file().await.unwrap();

            if new_config != config {
                tx.send(new_config.clone()).unwrap();
            }
            config = new_config;
        }
    });

    let mut handles = vec![];

    for _ in 0..5 {
        let mut rx = rx.clone();

        let handle = tokio::spawn(async move {
            let op = my_async_operation();
            tokio::pin!(op);

            let mut conf = rx.borrow().clone();

            let mut op_start = Instant::now();
            let sleep = time::sleep_until(op_start + conf.timeout);
            tokio::pin!(sleep);

            loop {
                tokio::select! {
                    _ = &mut sleep => {
                        op.set(my_async_operation());

                        op_start = Instant::now();

                        sleep.set(time::sleep_until(op_start + conf.timeout));
                    }
                    _ = rx.changed() => {
                        conf = rx.borrow_and_update().clone();

                        sleep.as_mut().reset(op_start + conf.timeout);
                    }
                    _ = &mut op => {
                        return
                    }
                }
            }
        });

        handles.push(handle);
    }

    for handle in handles.drain(..) {
        handle.await.unwrap();
    }
}
```

## 时间相关

### Sleep

```rust
use tokio::time::{self, sleep, sleep_until, Duration, Instant};

#[tokio::main]
async fn main() {
    sleep(Duration::from_micros(100)).await;
    println!("100 ms have elapsed");

    sleep_until(Instant::now() + Duration::from_micros(100)).await;

    let mut interval = time::interval(Duration::from_secs(1));
    interval.tick().await;
    interval.tick().await;
    interval.tick().await;
    interval.tick().await;

    let start = Instant::now() + Duration::from_secs(1);
    let mut interval = time::interval_at(start, Duration::from_secs(1));
    interval.tick().await;
    interval.tick().await;
    interval.tick().await;
    interval.tick().await;
}
```

### Interval

```rust
use tokio::time::{self, sleep, sleep_until, Duration, Instant};

#[tokio::main]
async fn main() {
    sleep(Duration::from_micros(100)).await;
    println!("100 ms have elapsed");

    sleep_until(Instant::now() + Duration::from_micros(100)).await;

    let mut interval = time::interval(Duration::from_secs(1));
    interval.tick().await;
    interval.tick().await;
    interval.tick().await;
    interval.tick().await;

    let start = Instant::now() + Duration::from_secs(1);
    let mut interval = time::interval_at(start, Duration::from_secs(1));
    interval.tick().await;
    interval.tick().await;
    interval.tick().await;
    interval.tick().await;
}
```

### Timeout

```rust
use tokio::{
    sync::oneshot,
    time::{timeout, Duration},
};

#[tokio::main]
async fn main() {
    let (_tx, rx) = oneshot::channel::<u32>();

    if let Err(_) = timeout(Duration::from_secs(1), rx).await {
        println!("did not receive value within 1 s");
    }
}
```
