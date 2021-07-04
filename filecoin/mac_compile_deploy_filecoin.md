# Mac 下部署 Filecoin源码

环境：Mac air M1 机器


## X-Code CLI tools

安装CommandLineTools

```bash

xcode-select -p

> /Library/Developer/CommandLineTools

```

如果没有安装，执行命令：

```bash

xcode-select --install

```


## brew 安装依赖包

```bash

brew install go bzr jq pkg-config rustup hwloc

```

 使用 homebrew 的时候，注意是 x86_64 还是 arm64 的架构 ，如果使用x86_64，那么其它环境，比如go,rust等都要保持一致，确保编译出来文件是统一的 CPU 架构。

 Mac m1 x86_64 指令集转换成 arm64指令集出处在哪？

[在 M1 芯片 Mac 上使用 Homebrew](https://zhuanlan.zhihu.com/p/335634215)


## 构建源码

### 克隆代码

```bash

git clone https://github.com/filecoin-project/lotus.git
cd lotus/
git checkout v1.10.0

```

### 更新子模块

```bash

git submodule update --init --recursive

```

注意：如果子模块更新不下来，可以手动下载

### 设置环境变量

```bash

export GOARCH=amd64
export CGO_ENABLED=1
export LIBRARY_PATH=/usr/local/lib
export FFI_BUILD_FROM_SOURCE=1

```

### 编译 filecoin-ffi 子模块

```bash

cd extern/filecoin-ffi
git fetch -a
git checkout master

make clean
make

```

### 编译 lotus

```bash

cd ../../
make lotus

```

## 安装

 ```bash

sudo make install

 ```

## 问题汇总


###  编译的 rust 版本不一致

找到filecoin-ffi 下的 rust-toolchain 文件，修改配置：

```

nightly-2021-06-30

```


### ld: library not found for -lhwloc


```bash

# arm64 架构
export LDFLAGS="-L/opt/homebrew/lib"
export CPPFLAGS="-I/opt/homebrew/include"
export CPATH=/opt/homebrew/include
export LIBRARY_PATH=/opt/homebrew/lib
export LD_LIBRARY_PATH=/opt/homebrew/lib

```

```bash

# intel x86_64 架构
export LDFLAGS="-L/usr/local/lib"
export CPPFLAGS="-I/usr/local/include"
export CPATH=/usr/local/include
export LIBRARY_PATH=/usr/local/lib
export LD_LIBRARY_PATH=/usr/local/lib

```

### building for macOS-arm64 but attempting to link with file built for macOS-x86_64

查看文件的cpu类型命令

```bash

lipo -info libfilcrypto.a

```

原因：编译过程中，使用了不一致的CPU指令集，使用同一套指令集编译即可。

[分不清ARM和X86架构，别跟我说你懂CPU](https://zhuanlan.zhihu.com/p/21266987)

[在 M1 芯片 Mac 上使用 Homebrew](https://zhuanlan.zhihu.com/p/335634215)


## git 下载慢


添加代理

```bash

git config --global http.proxy  'http://127.0.0.1:55403'

git config --global https.proxy  'http://127.0.0.1:55403'

```

移除代理

```bash

git config --global --unset http.proxy
git config --global --unset https.proxy

```

```bash

unset http_proxy
unset https_proxy

```

