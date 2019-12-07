---
date: 2018-12-25T14:20:00+08:00
title: hyper-balance
weight: 220
menu:
  main:
    parent: "lib"
description : "hyper-balance"
---



Instruments HTTP responses to drop handles when their first body message is received

在收到第一个正文消息时，  度量HTTP响应，以丢弃句柄

### 结构

PendingUntilFirstDataBody 和 PendingUntilEosBody 包裹一个 body，和处理这个body 的 handler，目的是在满足条件时 drop 掉 handler：

```rust
#[derive(Clone, Debug, Default)]
pub struct PendingUntilFirstData(());

#[derive(Clone, Debug, Default)]
pub struct PendingUntilEos(());

#[derive(Debug)]
pub struct PendingUntilFirstDataBody<T, B> {
    handle: Option<T>,
    body: B,
}

#[derive(Debug)]
pub struct PendingUntilEosBody<T, B> {
    handle: Option<T>,
    body: B,
}
```

### PendingUntilFirstData 的实现

实现 Instrument trait：

```rust
impl<T, B> Instrument<T, http::Response<B>> for PendingUntilFirstData
where
    B: Payload,
{
    type Output = http::Response<PendingUntilFirstDataBody<T, B>>;

    fn instrument(&self, handle: T, rsp: http::Response<B>) -> Self::Output {
        let (parts, body) = rsp.into_parts();
        let handle = if body.is_end_stream() {
            // 如果是 end of stream
            // drop handle 并设置handler为none
            drop(handle);
            None
        } else {
            Some(handle)
        };
        let body = PendingUntilFirstDataBody { handle, body };
        http::Response::from_parts(parts, body)
    }
}
```

原理就是将 handler 和 Response 绑定起来存储在 PendingUntilFirstDataBody，直到 end stream，就将 handler drop 并设置为 none。

TBD：first data 和 end stream 的意义不对。待和作者确认。

### PendingUntilEos

实现 Instrument trait：

```rust
impl<T, B> Instrument<T, http::Response<B>> for PendingUntilEos
where
    B: Payload,
{
    type Output = http::Response<PendingUntilEosBody<T, B>>;

    fn instrument(&self, handle: T, rsp: http::Response<B>) -> Self::Output {
        let (parts, body) = rsp.into_parts();
        let handle = if body.is_end_stream() {
            // 如果是 end of stream
            // drop handle 并设置handler为none
            drop(handle);
            None
        } else {
            Some(handle)
        };
        let body = PendingUntilEosBody { handle, body };
        http::Response::from_parts(parts, body)
    }
}
```

