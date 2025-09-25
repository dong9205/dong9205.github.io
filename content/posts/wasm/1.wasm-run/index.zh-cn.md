---
weight: 0
title: "编写WASM(WebAssembly)并运行"
subtitle: ""
date: 2024-01-24T16:27:41+08:00
lastmod: 2024-01-24T16:27:41+08:00
draft: false
summary: ""
license: ""
tags: ["wasm","go"]
resources:
- name: "featured-image"
  src: ""
categories: 
- "WASM"

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: false
  url: 
# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

<!--more-->

# 编写WASM(WebAssembly)并运行

## 编写WASM（WebAssembly）

### GO

#### 安装tinygo 

> https://github.com/tinygo-org/tinygo/releases

```bash
wget https://github.com/tinygo-org/tinygo/releases/download/v0.30.0/tinygo0.30.0.linux-amd64.tar.gz
tar xf tinygo0.30.0.linux-amd64.tar.gz -C /usr/local/
/usr/local/tinygo/bin/tinygo version
tinygo version 0.30.0 linux/amd64 (using go version go1.16.9 and LLVM version 16.0.1)
```

#### 暴露一个add函数

```go
package main

func main() {}

//export add
func add(a, b int) int {
	return a + b
}

```

#### build为wasm	

```bash
tinygo build -o go.wasm -target=wasi go/wasm.go

```

### rust

#### 安装wasm-pack
> 将代码编译为 WebAssembly，并生成正确的打包以供在浏览器中使用

```bash
cargo install wasm-pack
cargo install wasmer-cli
cargo install wasm-bindgen-cli
rustup target add wasm32-unknown-unknown
rustup target add wasm32-wasi
```

#### 创建一个webAssmebly包

`cargo new wasi-demo`


#### 修改src/main.rs

```rust
pub fn main() {
}

#[no_mangle]
pub extern fn add(left: usize, right: usize) -> usize {
    let sum = left + right;
    return sum;
}

```

#### 编译为wasm格式

```bash
wasm-pack build --target web

```

## 运行wasm

### Go运行

#### 编写wasmer-go运行代码 保存为`sample.go`文件
```go
package main

import (
	"flag"
	"fmt"
	"os"

	wasmer "github.com/wasmerio/wasmer-go/wasmer"
)

func main() {
	var wasmFile = flag.String("wasm_file", "go.wasm", "wasm file")
	flag.Parse()
	wasmBytes, _ := os.ReadFile(*wasmFile)

	store := wasmer.NewStore(wasmer.NewEngine())
	module, _ := wasmer.NewModule(store, wasmBytes)

	wasiEnv, _ := wasmer.NewWasiStateBuilder("wasi-program").
		// Choose according to your actual situation
		// Argument("--foo").
		// Environment("ABC", "DEF").
		// MapDirectory("./", ".").
		Finalize()
	importObject, err := wasiEnv.GenerateImportObject(store, module)
	check(err)

	instance, err := wasmer.NewInstance(module, importObject)
	check(err)

	start, err := instance.Exports.GetWasiStartFunction()
	check(err)
	start()

	add, err := instance.Exports.GetFunction("add")
	check(err)
	result, _ := add(1, 2)
	fmt.Println(result)
}

func check(e error) {
	if e != nil {
		panic(e)
	}
}

```

### 运行

```bash
# 测试GO生成的wasm
 $ go run sample.go --wasm_file ./go.wasm                                                                                                                                                                                                                                    
3

# 测试rust生成的wasm
 $ go run sample.go --wasm_file wasi-demo/target/wasm32-wasi/release/wasi-demo.wasm
3

```

## 参考文档
se
https://blog.csdn.net/weixin_47560078/article/details/130559636
https://github.com/wasmerio/wasmer-go/tree/master/examples/wasi
https://www.wkwkk.com/articles/1c90cd3673398f7f.html
