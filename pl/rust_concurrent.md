# Rust 并发编程

在许多编程语言中，基本都提供了并发编程的解决方案。常见的就是线程，当然，有的语言使用协程来实现并发编程。

## 线程

在rust中，使用thread::spawn 函数来并传入闭包来实现线程。

```rust

use std::thread;
use std::time::Duration;

fn main() {
     let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap();
}

```

其中，join会阻塞当前线程，直到该线程执行结束。


## 线程通信（通道）

在go语言中，使用chanel(管道)来实现协程之间的通信，而rust也是使用chanel实现线程间通信的。

mpsc 是 多个生产者，单个消费者（multiple producer, single consumer）的缩写。

```rust

use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}

```

tx : transmitter, 消息的发送者

rx : receiver, 表示消息的接收者

注意： move关键表示转移所有权到thread::spawn的线程中，因为在线程中访问了tx。(tx, rx) 这种数据类型表示元组，是一种组合数据类型。


## 互斥锁

rust语言中提供了互斥器的api，使用Mutex::new来创建互斥器。

```rust

use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    {
        let mut num = m.lock().unwrap();
        *num = 6;
    }

    println!("m = {:?}", m);
}

```

Mutex<T> 是一个智能指针。更准确的说，lock 调用 返回 一个叫做 MutexGuard 的智能指针。

如果要在多个线程直接共享变量，则需要使用原子引用计数器Arc<T>，使用如下：


```rust

use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
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

## 读写锁

rust中的读写锁，使用Arc::RwLock创建。RwLock允许同一时间有多个线程读，只存在一个线程写。读写锁可用于缓存的应用场景。

```rust

use std::thread;
use std::sync::{Arc, Mutex};
use std::time;
use std::collections::{HashMap};
use std::sync::RwLock;
#[macro_use]
extern crate lazy_static;

lazy_static! {
    static ref HASHMAP: HashMap<u32,String> = {
        let mut m = HashMap::new();
        m.insert(0, "foo".into());
        m.insert(1, "bar".into());
        m.insert(2, "baz".into());
        m
    };
}

fn main(){
    // 非全局式
    {
        let mut hp = HashMap::new();
        hp.insert("1".to_string(),1);
        let lock_1 = Arc::new(RwLock::new(hp));//多个线程读写
        {
            println!("single thread write =>");
            let mut w = lock_1.write().unwrap();
            (*w).insert("2".to_string(),2);
        }
        {
            println!(" single thread read =>");
            let r = lock_1.read().unwrap();
            println!("r:{:?} {:?}",1,*r);
        }
        {
            println!("multi  readers => ");
            for i in 0..10{
                let lock = lock_1.clone();
                thread::spawn(move ||{
                    let r = lock.read().unwrap();
                    println!("r:{:?} {:?}",i,*r);
                }); 
            } 
        }
        {
            println!("multi  writers => ");
            for i in 0..10{
                let lock = lock_1.clone();
                thread::spawn(move ||{
                    let mut _w = lock.write().unwrap();
                    (*_w).insert(i.to_string(),i*i);
                    println!(":{:?} {:?}",i,*_w);
                }); 
            } 

        }
    

    }
    println!("RwLock=> mode");
    //全局式

    {
        let lock_1 = RwLock::new(HASHMAP.clone());//一个写，多个读
        // .read(); .write()
        {
            println!("single thread write =>");
            let mut w = lock_1.write().unwrap();
            (*w).insert(2,"2".to_string());
            println!("*w =>{:?}",*w);
        }
        {
            println!(" single thread read =>");
            let r = lock_1.read().unwrap();
            println!("r:{:?} {:?}",1,r);
        }

    }

    std::thread::sleep(std::time::Duration::from_secs(10));    
}

```

最后，总结：在rust中，一旦代码可以编译了，就可以确认这些代码可以正确的运行于多线程环境，而不会出现其他语言中经常出现的那些难以追踪的 bug。所以，rust中的多线程编程，也叫做无畏并发。