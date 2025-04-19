---
title: "Rust并发集合"
date: {{ date }}
categories:
    - 语言
    - Rsut
    - 并发编程
---

## 摘要

Rust提供的常见集合类型。`Vec<T>`, `HashMap<K, V>`, `HashSet<T>`, `VecDeque<T>`, `LinkedList<T>`, `BTreeMap<<K, V>`, `BTreeSet<T>`

## 线程安全的Vec

```rust
use std::sync::{Arc, Mutex};

fn main() {
    let shared_vec = Arc::new(Mutex::new(Vec::new()));

    let mut handles = vec![];
    for i in 0..5 {
        let shared_vec = Arc::clone(&shared_vec);
        let handle = std::thread::spawn(move || {
            let mut vec = shared_vec.lock().unwrap();
            vec.push(i);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let final_vec = shared_vec.lock().unwrap();
    println!("Final Vec: {final_vec:?}");
}
```

## 线程安全的HashMap

```rust
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
    thread,
};

fn main() {
    let shared_map = Arc::new(Mutex::new(HashMap::new()));

    let mut handles = vec![];
    for i in 0..5 {
        let shared_map = shared_map.clone();
        let handle = thread::spawn(move || {
            let mut map = shared_map.lock().unwrap();
            map.insert(i, i * i);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let final_map = shared_map.lock().unwrap();
    println!("Final hashmap: {final_map:?}");
}
```

## dashmap

```rust
use std::sync::Arc;

use dashmap::DashMap;

fn main() {
    let map = Arc::new(DashMap::new());
    let mut handles = vec![];

    for i in 0..10 {
        let map = map.clone();
        handles.push(std::thread::spawn(move || {
            map.insert(i, i);
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("DashMap: {map:?}");
}
```

## cuckoofilter

## evmap

## arc-swap

