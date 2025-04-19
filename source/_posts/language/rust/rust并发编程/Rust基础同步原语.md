---
title: "Rust基础同步原语"
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

使用RWMutex可以提高读操作的并发性，因为多个线程可以同时读取数据，而写操作是互斥的，确保了数据的一致性。调用read方法获取读锁，调用write方法获取写锁。

```rust
use std::{
    sync::{Arc, RwLock},
    thread,
};

fn main() {
    let counter = Arc::new(RwLock::new(100));

    let mut read_handlers = vec![];

    for _ in 0..3 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let num = counter.read().unwrap();
            println!("Reader {:?}: {}", thread::current().id(), *num);
        });
        read_handlers.push(handle);
    }

    let write_handle = thread::spawn(move || {
        let mut num = counter.write().unwrap();
        *num += 1;
        println!(
            "Writer {:?}: Incremented counter to {}",
            thread::current().id(),
            *num
        );
    });

    for handle in read_handlers {
        handle.join().unwrap();
    }
    write_handle.join().unwrap();
}
```

## 一次初始化Once

Once是一个线程安全的单次初始化器，确保某个代码块只执行一次。使用场景：全局初始化，例如全局变量、加载配置等。懒加载：在需要时进行一次初始化。例如单利模式，使用Once可以实现线程安全的单利模式，确保某个对象在整个程序生命周期中只初始化一次。

```rust
use std::sync::Once;

static INIT: Once = Once::new();

fn main() {
    INIT.call_once(|| {
        println!("Initialization code executed");
    });

    INIT.call_once(|| {
        println!("This won't be printed.");
    });
}
```

```rust
use std::sync::Once;

static INIT: Once = Once::new();
static mut GLOBAL_CONFIG: Option<String> = None;

fn init_global_config() {
    unsafe { GLOBAL_CONFIG = Some("Initialized global configuration".to_string()) };
}

fn get_global_config() -> &'static str {
    INIT.call_once(|| {
        init_global_config();
    });
    unsafe { GLOBAL_CONFIG.as_ref().unwrap() }
}

fn main() {
    println!("{}", get_global_config());
    println!("{}", get_global_config());
}
```

OnceCell是一个针对某种数据类型进行包装的懒加载容器，可以在需要执行一次性初始化，并在之后提供对初始化值的访问。OnceLock是一个可用于线程安全的懒加载的原语，类似于OnceCell，但是更简单，只能存储Copy类型的数据。

```rust
use std::sync::OnceLock;

static CELL: OnceLock<String> = OnceLock::new();

fn main() {
    std::thread::spawn(|| {
        let value = CELL.get_or_init(|| "hello, world".to_string());
        assert_eq!(value, "hello, world");
    })
    .join()
    .unwrap();

    let value = CELL.get();
    assert!(value.is_some());
    assert_eq!(value.unwrap().as_str(), "hello, world");
}
```

## 屏障/栅栏Barrier

Barrier是Rust标准库中的一种并发原语，用于多个线程之间创建一个同步点，他允许多个线程在某个点上等待，直到所有线程都到达该点。Barrier可以用于实现一些并行算法，例如并行排序、并行搜索等。

```rust
use std::{
    sync::{Arc, Barrier},
    time::Duration,
};

fn main() {
    let barrier = Arc::new(Barrier::new(3));

    let mut handles = vec![];

    for id in 0..3 {
        let barrier = Arc::clone(&barrier);
        let handle = std::thread::spawn(move || {
            println!("Thread {} working", id);
            std::thread::sleep(Duration::from_secs(id as u64));

            barrier.wait();

            println!("Thread {} resumed", id);
        });

        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

循环使用Barrier，可以实现一个简单的生产者-消费者模型。

```rust
use std::{
    sync::{Arc, Barrier},
    time::Duration,
};

use rand::Rng;

fn main() {
    let barrier = Arc::new(Barrier::new(10));
    let mut handles = vec![];

    for _ in 0..10 {
        let barrier = Arc::clone(&barrier);
        handles.push(std::thread::spawn(move || {
            println!("before wait1");
            let dur = rand::thread_rng().gen_range(100..1000);
            std::thread::sleep(Duration::from_millis(dur));

            barrier.wait();

            println!("after wait1");
            std::thread::sleep(Duration::from_secs(1));

            barrier.wait();
            println!("after wait2");
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

## 条件变量Condvar

Condvar是一个条件变量，用于在多个线程之间进行同步和通信。Condvar可以与Mutex一起使用，以实现线程之间的同步和通信。

```rust
use std::{
    sync::{Arc, Condvar, Mutex},
    time::Duration,
};

fn main() {
    let mutex = Arc::new(Mutex::new(false));
    let condvar = Arc::new(Condvar::new());

    let mut handles = vec![];

    for id in 0..3 {
        let mutex = Arc::clone(&mutex);
        let condvar = Arc::clone(&condvar);

        let handle = std::thread::spawn(move || {
            let mut guard = mutex.lock().unwrap();

            while !*guard {
                guard = condvar.wait(guard).unwrap();
            }

            println!("Thread {} woke up", id);
        });

        handles.push(handle);
    }

    std::thread::sleep(Duration::from_secs(2));

    {
        let mut guard = mutex.lock().unwrap();
        *guard = true;
        condvar.notify_all();
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

## mpsc

mpsc是Rust标准库提供的一个模块，提供了多生产者、单消费者的消息传递通道。

```rust
use std::{sync::mpsc::channel, thread};

fn main() {
    let (tx, rx) = channel();
    thread::spawn(move || {
        tx.send(10).unwrap();
    });
    assert_eq!(rx.recv().unwrap(), 10);
}
```

多生产者但消费者的例子

```rust
use std::{sync::mpsc::channel, thread};

fn main() {
    let (tx, rx) = channel();
    for i in 0..10 {
        let tx = tx.clone();
        thread::spawn(move || {
            tx.send(i).unwrap();
        });
    }

    for _ in 0..10 {
        let j = rx.recv().unwrap();
        println!("{}", j);
    }
}
```

```rust
use std::{
    sync::mpsc::sync_channel,
    thread,
};

fn main() {
    let (tx, rx) = sync_channel(3);

    for _ in 0..3 {
        let tx = tx.clone();
        thread::spawn(move || {
            tx.send("ok").unwrap();
        });
    }

    drop(tx);

    while let Ok(msg) = rx.recv() {
        println!("{msg}");
    }

    println!("completed");
}
```


## 原子操作atomic


```rust
use std::sync::atomic::{AtomicI64, Ordering};

fn main() {
    let atomic_num = AtomicI64::new(0);

    // 原子加载
    let num = atomic_num.load(Ordering::Relaxed);
    println!("{num}");

    // 原子加法并返回旧值
    let old = atomic_num.fetch_add(10, Ordering::SeqCst);
    println!("{old}");

    // 原子比较并交换
    let res = atomic_num.compare_exchange(10, 100, Ordering::Acquire, Ordering::Relaxed);
    println!("{res:?}");

    // 原子交换
    let swapped = atomic_num.swap(200, Ordering::Release);
    println!("{swapped}");

    // 原子存储
    atomic_num.store(10000, Ordering::Relaxed);
}
```

### 原子操作的Ordering

- Ordering::Relaxed：最轻量级的内存屏障，没有对执行顺序进行强制排序。允许编译器和处理器在原子操作周围进行指令重排。
- Ordering::Acquire：插入一个获取内存屏障，防止后续的读操作被重排序到当前操作之前。确保当前操作之前的所有读取操作都在当前操作之前执行。
- Ordering:Release：插入一个释放内存屏障，防止之前的写操作被重排序到当前操作之后。确保当前操作之后的所有操作都在当前操作之后执行。
- Ordering:AcqRel：插入一个获取释放内存屏障，既确保当前操作之前的所有读取操作都在当前操作之前执行，又确保之前的所有写操作都在当前操作之后执行。这种内存屏障提供了一种平衡，适用于某些获取和释放操作交替进行的场景。
- Ordering::SeqCst：插入一个全序内存屏障，保障所有线程都能看到一致的操作顺序。是最强的内存顺序，用于实现全局同步。


### Ordering::Relaxed

```rust
use std::{
    sync::{
        atomic::{AtomicBool, Ordering},
        Arc,
    },
    thread,
};

fn main() {
    let atomic_bool = Arc::new(AtomicBool::new(false));

    let producer_thread = {
        let atomic_bool = atomic_bool.clone();
        thread::spawn(move || {
            // 这里可能有指令重排
            atomic_bool.store(true, Ordering::Relaxed);
        })
    };

    let consumer_thread = {
        let atomic_bool = atomic_bool.clone();
        thread::spawn(move || {
            let value = atomic_bool.load(Ordering::Relaxed);
            println!("Received value: {}", value);
        })
    };

    producer_thread.join().unwrap();
    consumer_thread.join().unwrap();
}
```

### Ordering::Release

```rust
use std::{
    sync::{
        atomic::{AtomicBool, Ordering},
        Arc,
    },
    thread,
};

fn main() {
    let atomic_bool = Arc::new(AtomicBool::new(false));

    let producer_thread = {
        let atomic_bool = atomic_bool.clone();
        thread::spawn(move || {
            atomic_bool.store(true, Ordering::Release);
        })
    };

    let consumer_thread = {
        let atomic_bool = atomic_bool.clone();
        thread::spawn(move || {
            // 等待直到读取到布尔值为true
            while !atomic_bool.load(Ordering::Acquire) {
                // 这里可能进行自旋，知道读取到Acquire顺序的布尔值。
                // 注意：在实际应用中，可能使用高级的同步源于而不是使用自旋。
            }
            println!("Received value");
        })
    };

    producer_thread.join().unwrap();
    consumer_thread.join().unwrap();
}
```

这里例子中，生产者线程使用store方法将布尔值设置为true，而消费者线程使用load方法等待并读取的状态。由于使用了Ordering::Release，在生产者线程设置布尔值之后，会插入释放内存屏障，确保之前的所有操作都在操作之后执行。

### Ordering::AcqRel

### Ordering::SeqCst

## 理解内存顺序 Ordering

理解内存顺序（Ordering）是并发编程中最具挑战性的部分之一。让我尝试用更直观的方式来解释。

### 为什么不能都用 `Relaxed`？

如果你在两个线程中都使用 `Relaxed`：

```rust
// 生产者
atomic_bool.store(true, Ordering::Relaxed);

// 消费者
while !atomic_bool.load(Ordering::Relaxed) {}
```

这种情况下：
1. **没有同步保证**：编译器/CPU可能会重排序这些操作
2. **消费者可能永远看不到变化**：理论上，消费者线程可能永远读取到 `false`（虽然实际硬件上不太可能）
3. **没有 happens-before 关系**：无法保证生产者存储前的操作对消费者可见

### Release-Acquire 的作用

`Release` 和 `Acquire` 创建了一个 **同步关系**：

```
[生产者线程]
操作A
操作B
store(true, Release)  // 释放屏障 - 确保A和B在此之前完成

[消费者线程]
load(Acquire)         // 获取屏障 - 确保看到Release前的所有操作
操作C
操作D
```

关键点：
1. **Release 存储**：确保所有**之前的**写操作在原子存储之前完成
2. **Acquire 加载**：确保所有**之后的**读操作在原子加载之后执行
3. **同步点**：当 Acquire 加载看到 Release 存储的值时，就建立了 happens-before 关系

### 为什么消费者"仍然"能看到旧值？

你观察到的现象是正确的 - 消费者线程在开始时可能仍然看到 `false`，这不是内存顺序的问题，而是因为：

1. **时间问题**：生产者可能还没有执行存储操作
2. **这不是内存顺序要解决的问题**：内存顺序解决的是"看到值后能保证什么"，而不是"何时能看到值"

### 各 Ordering 级别的比较

| Ordering   | 保证程度 | 性能 | 适用场景 |
|------------|---------|------|----------|
| Relaxed    | 只保证原子性 | 最高 | 计数器等不需要同步的场景 |
| Release    | 释放语义 | 中 | 生产者发布数据 |
| Acquire    | 获取语义 | 中 | 消费者获取数据 |
| AcqRel     | 两者结合 | 中 | 读-修改-写操作 |
| SeqCst     | 完全顺序一致性 | 最低 | 需要最强保证的场景 |

### 实际建议

1. **优先使用高级同步原语**：如 `Mutex`、`channel`，它们已经正确实现了内存顺序
2. **只在必要时使用原子操作**：当你确实需要构建自定义同步机制时
3. **Relaxed 的合理使用**：适合简单的标志位或计数器，当值的可见性延迟不重要时

记住：**内存顺序不是关于操作何时发生，而是关于当操作对其他线程可见时，能保证什么**。