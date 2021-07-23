# linux　openethereum 源码编译


## 安装依赖

```bah

sudo apt-get install build-essential cmake libudev-dev

```

## 安装 rust

```bash

curl https://sh.rustup.rs -sSf | sh

```

## 下载源码

```
git clone https://github.com/openethereum/openethereum.git

```

## 编译

```bash

cargo build

```

## 问题

```
error[E0061]: this function takes 1 argument but 2 arguments were supplied
   --> /home/xingzjx/.cargo/registry/src/github.com-1ecc6299db9ec823/logos-derive-0.7.7/src/lib.rs:55:20
    |
55  |             extras.insert(util::ident(&ext), |_| panic!("Only one #[extras] attribute can be declared."));
    |                    ^^^^^^ -----------------  ----------------------------------------------------------- supplied 2 arguments
    |                    |
    |                    expected 1 argument
    |
note: associated function defined here
   --> /home/xingzjx/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/option.rs:850:12
    |
850 |     pub fn insert(&mut self, value: T) -> &mut T {
    |            ^^^^^^

```

跟踪源码位置

logos-derive-0.7.7/src/lib.rs

```rust
pub fn logos(input: TokenStream) -> TokenStream {
    let item: ItemEnum = syn::parse(input).expect("#[token] can be only applied to enums");

    let size = item.variants.len();
    let name = &item.ident;

    let mut extras = None;
    let mut error = None;
    let mut end = None;

    for attr in &item.attrs {
        if let Some(ext) = value_from_attr("extras", attr) {
            extras.insert(util::ident(&ext), |_| panic!("Only one #[extras] attribute can be declared."));
        }
    }

    // Initially we pack all variants into a single fork, this is where all the logic branching
    // magic happens.
    let mut fork = Fork::default();

    // 省略　....

}

```

stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/option.rs

```rust

#[inline]
#[stable(feature = "option_insert", since = "1.53.0")]
pub fn insert(&mut self, value: T) -> &mut T {
    *self = Some(value);

    match self {
        Some(v) => v,
        // SAFETY: the code above just filled the option
        None => unsafe { hint::unreachable_unchecked() },
    }
}

```

通过代码可以知道，是版本兼容的问题。当前版本号是 1.55.0，　尝试降低版本到　１.52.0

```bash

rustup install １.52.0

```

查看可以获取的版本

```bash

ls ~/.rustup/toolchains


```

切换版本

```bash

rustup default 1.52.0

```

然后，重新编译，编译通过。
