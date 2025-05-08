# send和sync的区别

`Send` 和 `Sync` 是 Rust 用来描述并发安全的两个标记 trait，它们控制如何在多线程环境中共享数据。它们都通过 Rust 编译器的类型系统来确保线程安全，但它们检查的并发安全性点不同。

### 1. **`Send` 和 `Sync` 的定义**

- **`Send`**：标记一个类型的所有权可以在线程之间传递。
- **`Sync`**：标记一个类型的引用可以在多个线程之间共享。

#### `Send`:
- 如果一个类型实现了 `Send`，那么该类型的所有权可以被安全地传递到另一个线程中。换句话说，一个类型的所有权是可以在不同线程之间移动的。
- 实现 `Send` 的类型，能够安全地传递给其他线程，而不担心数据竞争的问题。对于大多数类型来说，所有权可以传递，但某些类型（例如 `Rc<T>`）因为是非线程安全的，所以不实现 `Send`。

#### `Sync`:
- 如果一个类型实现了 `Sync`，那么该类型的引用可以在多个线程中共享。更具体地说，`Sync` 是一个标记 trait，表示该类型的 **不可变引用** 可以安全地在线程之间共享。
- 如果一个类型 `T` 实现了 `Sync`，则类型 `&T` 可以被多个线程同时共享。而对于可变引用，Rust 要求使用同步原语（如 `Mutex` 或 `RwLock`）来确保线程安全。

### 2. **`Send` 和 `Sync` 检查的区别**

#### 1. **`Send` 的检查**

`Send` 的主要检查点在于 **所有权的转移**。Rust 编译器会检查类型是否允许将其所有权从一个线程转移到另一个线程。在大多数情况下，如果类型中没有包含不能在多个线程间安全移动的字段，`Send` 会自动实现。

- **不能实现 `Send` 的类型**：某些类型由于它们内部的状态是线程不安全的，因此不能在多个线程之间传递。例如，`Rc<T>` 是一个单线程引用计数类型，因此它不实现 `Send`，因为它无法跨线程共享引用计数。
- **可以实现 `Send` 的类型**：如果一个类型的所有字段都是 `Send` 的，那么这个类型也可以实现 `Send`。比如 `Arc<T>`，由于 `T` 是线程安全的（例如 `T` 里面是 `Sync` 类型的），因此 `Arc<T>` 是 `Send` 的。

**编译器的检查：**
- 如果一个类型包含无法在线程之间安全传递的类型（如 `Rc<T>`），编译器会报错。
- 编译器检查类型是否在多个线程之间传递其所有权时，是否存在非线程安全的类型。

**例子：**

```rust
use std::sync::Arc;
use std::thread;

struct Data {
    value: i32,
}

fn main() {
    let data = Arc::new(Data { value: 42 });
    let handle = thread::spawn(move || {
        println!("{}", data.value);
    });
    handle.join().unwrap();  // `Arc<Data>` 实现了 `Send`
}
```

在这个例子中，`Arc<T>` 是 `Send` 的，因此 `data` 可以安全地传递给新线程。

#### 2. **`Sync` 的检查**

`Sync` 的主要检查点在于 **引用的共享**。Rust 编译器检查类型是否允许其不可变引用（`&T`）在线程间共享。实现 `Sync` 的类型允许多个线程同时持有对该类型的不可变引用。

- **不能实现 `Sync` 的类型**：如果一个类型允许对其内部数据进行可变访问而没有同步机制，编译器就不会让它实现 `Sync`。例如，`Cell<T>` 和 `RefCell<T>` 就不实现 `Sync`，因为它们允许在多个线程中进行可变借用，这在并发环境中是危险的。
- **可以实现 `Sync` 的类型**：如果一个类型本身是不可变的，或者它包含线程安全的字段（例如通过 `Mutex` 或 `RwLock`），那么它是可以实现 `Sync` 的。比如，`Arc<Mutex<T>>` 和 `RwLock<T>` 都是实现了 `Sync` 的。

**编译器的检查：**
- 如果一个类型包含可变数据，并且没有适当的同步原语（如 `Mutex` 或 `RwLock`），编译器会阻止该类型实现 `Sync`。
- 如果你试图共享一个类型的不可变引用，但该类型的内部状态允许并发的可变访问，编译器会报错。

**例子：**

```rust
use std::sync::Arc;
use std::sync::Mutex;
use std::thread;

struct Counter {
    value: i32,
}

impl Counter {
    fn new() -> Self {
        Counter { value: 0 }
    }

    fn increment(&mut self) {
        self.value += 1;
    }

    fn get_value(&self) -> i32 {
        self.value
    }
}

fn main() {
    let counter = Arc::new(Mutex::new(Counter::new()));

    let mut handles = vec![];

    for _ in 0..10 {
        let counter_clone = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut counter = counter_clone.lock().unwrap();
            counter.increment();
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let counter = counter.lock().unwrap();
    println!("Final counter value: {}", counter.get_value());
}
```

在这个例子中，`Mutex<T>` 和 `Arc<T>` 都实现了 `Sync`，因此可以在线程间共享不可变引用。

### 3. **总结**

| 语义   | 检查点 | 描述                                                         | 例子                        |
|--------|--------|--------------------------------------------------------------|-----------------------------|
| **`Send`** | 类型的所有权是否可以跨线程传递 | 确保一个类型的所有权可以在多个线程之间转移。 | `Arc<T>`（可传递所有权） |
| **`Sync`** | 类型的引用是否可以跨线程共享 | 确保多个线程可以安全地共享该类型的不可变引用。 | `Arc<Mutex<T>>`, `Arc<RwLock<T>>`（可共享引用） |

- **`Send`** 关注的是 **所有权的转移**，它检查类型的所有权是否可以安全地在线程间移动。
- **`Sync`** 关注的是 **引用的共享**，它检查类型是否可以在多个线程中共享不可变引用。

编译器通过这些语义来确保 Rust 程序中的并发安全，并强制类型系统遵循严格的线程安全规则，避免数据竞争、死锁等并发问题。