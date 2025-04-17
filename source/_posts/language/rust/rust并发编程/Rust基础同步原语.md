---
title: "Rust容器同步原语"
date: {{ date }}
categories:
    - 语言
    - Rsut
    - 并发编程
---

## 摘要

继续学习同步原语。常见的同步原语有互斥锁、信号量、条件变量等。

## Arc

Rust的Arc代表原子引用计数，是一种用于多线程环境的智能指针。它允许多个地方共享数据。

```rust
use std::{sync::Arc, thread};

fn main() {
    let data = Arc::new(100);

    let thread1 = {
        let data = Arc::clone(&data);

        thread::spawn(move || {
            println!("thread1: {}", data);
        })
    };

    let thread2 = {
        let data = Arc::clone(&data);

        thread::spawn(move || {
            println!("thread2: {}", data);
        })
    };

    thread1.join().unwrap();
    thread2.join().unwrap();
}
```

## 互斥锁Mutex

### Lock

当需要再多个线程中共享可变数据时，常常会结合使用Arc和Mutex。

```rust
use std::{
    sync::{Arc, Mutex},
    thread,
};

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let mut handles = vec![];

    for _ in 0..5 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

### try_lock

Mutex的try_lock方法尝试获取锁，如果锁已经被其他线程持有，则立即返回Err而不是阻塞线程。

```rust
use std::{
    sync::{Arc, Mutex},
    thread,
};

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let mut handlers = vec![];

    for _ in 0..100 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            if let Ok(mut num) = counter.try_lock() {
                *num += 1;
            } else {
                println!("Thread failed to acquire lock.");
            }
        });
        handlers.push(handle);
    }

    for handler in handlers {
        handler.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

### Poisoning

在Rust中，poisoning是一种用于处理线程panic导致的不可恢复的状态的机制。当一个线程持有锁的时候panic，锁会被标记为poisoned，此后任何线程尝试获取这个锁时，都会得到一个PoisonError，它包含了一个标识锁是否被poisoned的标志。这样线程可以检测到之前的panic，并进行相应的处理。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // 创建一个共享的 Mutex
    let data = Arc::new(Mutex::new(42));

    // 克隆 Arc 用于新线程
    let data_clone = Arc::clone(&data);

    // 创建一个线程，在持有锁时 panic
    let handle = thread::spawn(move || {
        let mut num = data_clone.lock().unwrap();
        *num += 1;

        // 这里故意 panic，导致 Mutex 中毒
        panic!("Oops, something went wrong!");

        // 正常情况下，锁会在作用域结束时释放
    });

    // 等待线程结束（会 panic）
    let _ = handle.join();

    // 尝试在主线程中获取锁
    match data.lock() {
        Ok(guard) => {
            println!("Lock acquired successfully: {}", *guard);
        }
        Err(poisoned) => {
            // 锁已中毒，但我们仍然可以访问数据
            let guard = poisoned.into_inner();
            println!(
                "Mutex was poisoned, but we can still recover the data: {}",
                *guard
            );
        }
    };
}
```

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    // 创建一个共享的 Mutex
    let data = Arc::new(Mutex::new(0));

    let mut handles = vec![];

    // 创建5个工作线程，正常递增计数器
    for i in 0..5 {
        let data_clone = Arc::clone(&data);
        let handle = thread::spawn(move || {
            // 模拟一些工作
            thread::sleep(Duration::from_millis(100 * i));

            let mut num = data_clone.lock().unwrap();
            *num += 1;
            println!("Thread {} incremented to {}", i, *num);
        });
        handles.push(handle);
    }

    // 创建一个会 panic 的线程
    let data_clone = Arc::clone(&data);
    let panic_handle = thread::spawn(move || {
        thread::sleep(Duration::from_millis(250));

        let mut num = data_clone.lock().unwrap();
        *num += 1;
        println!("Panic thread incremented to {}", *num);

        // 这里故意 panic，导致 Mutex 中毒
        panic!("Panic thread crashed while holding the lock!");
    });
    handles.push(panic_handle);

    // 再创建3个工作线程，可能会遇到中毒情况
    for i in 5..8 {
        let data_clone = Arc::clone(&data);
        let handle = thread::spawn(move || {
            thread::sleep(Duration::from_millis(350 + 100 * (i - 5)));

            // 尝试获取锁，处理可能的中毒情况
            match data_clone.lock() {
                Ok(mut num) => {
                    *num += 1;
                    println!("Thread {} incremented to {}", i, *num);
                }
                Err(poisoned) => {
                    let mut num = poisoned.into_inner();
                    *num += 1;
                    println!(
                        "Thread {} recovered from poison, incremented to {}",
                        i, *num
                    );
                }
            }
        });
        handles.push(handle);
    }

    // 等待所有线程完成
    for handle in handles {
        // 忽略线程 panic 的错误
        let _ = handle.join();
    }

    // 最终检查 Mutex 状态
    match data.lock() {
        Ok(guard) => {
            println!("Final value (no poison): {}", *guard);
        }
        Err(poisoned) => {
            let guard = poisoned.into_inner();
            println!("Final value (recovered from poison): {}", *guard);
        }
    };
}
```

### 更快的释放互斥锁

```rust
use std::sync::{Arc, Mutex};

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let mut handles = vec![];

    for _ in 0..5 {
        let counter = Arc::clone(&counter);
        let handle = std::thread::spawn(move || {
            {
                let mut num = counter.lock().unwrap();
                *num += 1;
            }
            // 这里锁已经释放，可以继续使用 counter
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Final count: {}", *counter.lock().unwrap());
}
```

```rust
use std::sync::{Arc, Mutex};

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let mut handles = vec![];

    for _ in 0..5 {
        let counter = Arc::clone(&counter);
        let handle = std::thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;

            // 手动释放锁
            drop(num);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Final count: {}", *counter.lock().unwrap());
}
```

## 读写锁RWMutex

## 一次初始化Once

## 屏障/栅栏Barrier

## 条件变量Condvar

## LazyCell和LazyLock

## Exclusive

## mpsc

## 信号量Semaphore

## 原子操作atomic

### 原子操作的Ordering

### Ordering::Relaxed

### Ordering::Release

### Ordering::AcqRel

### Ordering::SeqCst
