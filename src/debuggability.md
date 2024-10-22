# 可调试性

## 所有公共类型实现 `Debug` (C-DEBUG)

几乎没有例外情况。

## `Debug` 表示不为空 (C-DEBUG-NONEMPTY)

即使对于概念上为空的值，`Debug` 表示也不应为空。

```rust
let empty_str = "";
assert_eq!(format!("{:?}", empty_str), "\"\"");

let empty_vec = Vec::<bool>::new();
assert_eq!(format!("{:?}", empty_vec), "[]");
```

### 说明

确保所有公共类型都实现 `Debug`，以便于开发者在调试时查看类型的内部状态。即使是空的或简单的类型，其 `Debug` 输出也应提供足够的信息来表示其状态。