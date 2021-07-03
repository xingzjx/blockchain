git config --global http.proxy  'http://127.0.0.1:55403'
git config --global https.proxy  'http://127.0.0.1:55403'
 
unset http_proxy
unset https_proxy

git config --global --unset http.proxy
git config --global --unset https.proxy


问题：编译的 rust 版本不一致

Rust-toolchain
nightly-2021-06-30


问题：ld: library not found for -lhwloc

export LDFLAGS="-L/opt/homebrew/lib"
export CPPFLAGS="-I/opt/homebrew/include"
export CPATH=/opt/homebrew/include
export LIBRARY_PATH=/opt/homebrew/lib
export LD_LIBRARY_PATH=/opt/homebrew/lib

intel x86架构
export LDFLAGS="-L/usr/local/lib"
export CPPFLAGS="-I/usr/local/include"
export CPATH=/usr/local/include
export LIBRARY_PATH=/usr/local/lib
export LD_LIBRARY_PATH=/usr/local/lib



问题：building for macOS-arm64 but attempting to link with file built for macOS-x86_64

在 M1 芯片 Mac 上使用 Homebrew
https://zhuanlan.zhihu.com/p/335634215

查看文件的cpu类型
lipo -info libfilcrypto.a

file /bin/bash

make ARCH=arm64