---
date: 2018-12-25T14:20:00+08:00
title: futures-mpsc-lossy
weight: 210
menu:
  main:
    parent: "lib"
description : "futures-mpsc-lossy"
---



创建一个有损的channel，支持多生产者/单消费者。

该channel是有界的，但没有提供背压(backpressure)机制。 虽然它返回了它无法接受的内容，但它不会通知生产者容量可用性。

这允许生产者在此channel上发送事件，而不用获得对发件人的可变引用。

### 结构

Receiver 、Sender 各自包装一个 `mpsc::UnboundedReceiver<T>` 和 mpsc::UnboundedSender<T>，带有capacity信息：

```rust
pub struct Receiver<T> {
    rx: mpsc::UnboundedReceiver<T>,
    capacity: Arc<AtomicUsize>,
}

pub struct Sender<T> {
    tx: mpsc::UnboundedSender<T>,
    capacity: Arc<AtomicUsize>,
}
```

Arc 是 “Atomically Reference Counted”，线程安全的 reference-counting pointer。

channel 方法创建 Receiver 、Sender ：

```rust
pub fn channel<T>(capacity: usize) -> (Sender<T>, Receiver<T>) {
	// mpsc = multi-producer, single-consumer
    let (tx, rx) = mpsc::unbounded();
    let capacity = Arc::new(AtomicUsize::new(capacity));

    let s = Sender {
        tx,
        capacity: capacity.clone(),
    };

    let r = Receiver { rx, capacity };

    (s, r)
}
```

定义 SendError

```rust
#[derive(Copy, Clone, Debug, Eq, PartialEq)]
pub enum SendError<T> {
    NoReceiver(T),
    Rejected(T),
}
```

###　Receiver实现

Receiver 的 Stream trait 实现：

```rust
impl<T> Stream for Receiver<T> {
    type Item = T;
    type Error = ();

    fn poll(&mut self) -> Poll<Option<T>, Self::Error> {
        // 调用 Stream 的 poll（）
        match self.rx.poll() {
            // 如果 feature 准备完成，就 capacity + 1
            Ok(Async::Ready(Some(v))) => {
                self.capacity.fetch_add(1, Ordering::SeqCst);
                Ok(Async::Ready(Some(v)))
            }
            res => res,
        }
    }
}
```

debug trait 实现：

```rust
impl<T> fmt::Debug for Receiver<T> {
    fn fmt(&self, fmt: &mut fmt::Formatter) -> fmt::Result {
        fmt.debug_struct("Receiver")
            .field("capacity", &self.capacity)
            .finish()
    }
}
```

### Sender实现

Sender 的 lossy_send 实现：

```rust
impl<T> Sender<T> {
    pub fn lossy_send(&self, v: T) -> Result<(), SendError<T>> {
        loop {
            let cap = self.capacity.load(Ordering::SeqCst);
            if cap == 0 {
                // 如果容量为0，则拒绝发送
                return Err(SendError::Rejected(v));
            }

            // CAS 设置容量减1
            let ret = self
                .capacity
                .compare_and_swap(cap, cap - 1, Ordering::SeqCst);
            if ret == cap {
                break;
            }
        }

        self.tx
            .unbounded_send(v)
            .map_err(|se| SendError::NoReceiver(se.into_inner()))
    }
}
```

Sender 的 Sink 实现：

```rust
/// Drops events instead of exerting backpressure
impl<T> Sink for Sender<T> {
    type SinkItem = T;
    type SinkError = SendError<T>;

    fn start_send(&mut self, item: T) -> StartSend<Self::SinkItem, Self::SinkError> {
        // 即使被拒绝或者出错，也当成成功
        self.lossy_send(item).map(|_| AsyncSink::Ready)
    }

    fn poll_complete(&mut self) -> Poll<(), Self::SinkError> {
        Ok(().into())
    }
}
```

TBD: 如果 lossy_send 方法真的报错，capacity 的计算就会有bug。有机会去确认一下。

