---
date: 2018-12-25T14:20:00+08:00
title: never
weight: 230
menu:
  main:
    parent: "lib"
description : "never"
---

A type representing a value that can never materialize.

This would be `!`, but it isn't stable yet.

表示永远不会实现的值的类型。

这将是`！`，但它还不稳定。

```rust
#[derive(Clone, Copy, Debug, PartialEq)]
pub enum Never {}

impl fmt::Display for Never {
    fn fmt(&self, _: &mut fmt::Formatter) -> fmt::Result {
        match *self {}
    }
}

impl Error for Never {}
```

