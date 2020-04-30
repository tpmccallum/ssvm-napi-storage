# Second State WebAssembly VM for Node.js Addon

The [Second State VM (SSVM)](https://github.com/second-state/ssvm) is a high performance WebAssembly runtime optimized for server side applications. This project provides support for accessing SSVM as a Node.js addon. It allows Node.js applications to call WebAssembly functions written in Rust or other high performance languages. [Why do you want to run WebAssembly on the server-side?](https://cloud.secondstate.io/server-side-webassembly/why) The SSVM addon could interact with the wasm files generated by the [ssvmup](https://github.com/second-state/ssvmup) compiler tool.

## NOTICE

SSVM Node.js Addon is in active development.

In current stage, our prebuilt version **only supports** `Boost 1.65.1` and x64 Linux.
If you want to use other version of Boost library, you could use `--build-from-source` flag during addon installation.


## Setup for Rust, Nodejs, and ssvmup

```
$ apt-get update
$ apt install -y build-essential

$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
$ source $HOME/.cargo/env

$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash

$ nvm install v10.19.0
$ nvm use v10.19.0

$ npm i -g ssvmup
```


## Create new project

```
cargo new --lib hello
cd hello
```

## Change the cargo config file

Add the following to the `Cargo.toml` file.

```
[lib]
name = "hello"
path = "src/lib.rs"
crate-type =["cdylib"]

[dependencies]
wasm-bindgen = "0.2.50"
```

## Write Rust code

Below is the entire content of the `src/lib.rs` file.

```
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn say(s: String) -> String {
  let r = String::from("hello ");
  return r + &s;
}

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    return a + b;
}

#[wasm_bindgen]
pub fn reverse(v: Vec<u8>) -> Vec<u8> {
    let mut r = v.clone();
    r.reverse();
    return r;
}
```

In addition to the above code, functions like the example below can deliberately parse incoming JSON string data. This is an additional say example which uses valid JSON instead of just plain string.
```rust
use serde_json::json;
use wasm_bindgen::prelude::*;
use serde_json::{Result, Value};

#[wasm_bindgen]
pub fn say_with_json(s: String) -> String {
    let json_as_value: Value = serde_json::from_str(&s).unwrap();
    let first_word = String::from("Hello ");
    let concatenation = first_word + &serde_json::to_string(&json_as_value["name"]).unwrap();
    let response_object = json!({ "result": concatenation });
    return serde_json::to_string(&response_object).unwrap();
}
```
When given `{"name": "Tim"}` this `say_with_json` function returns `Hello Tim` wrapped in a response object (as valid JSON) like this `{"ssvm_response": ["{\"result\": \"Hello Tim\"}"]}`

## Build the WASM bytecode

```
ssvmup build
```

After building, our target wasm file is located at `pkg/hello_bg.wasm`.

## Requirements for SSVM addon

```
apt update
apt install -y build-essential libboost-all-dev
```

## Install SSVM addon for your application

```
npm install ssvm
```

or if you want to build from source:

```
npm install --build-from-source https://github.com/second-state/ssvm-napi
```

## Use SSVM addon

After installing SSVM addon, we could now interact with `hello_bg.wasm` generated by wasm-pack in Node.js.
Make sure you use the corresponding vm method to rust return type.

- Create a new folder at any path you want. (e.g. `mkdir application`)
- Copy `hello_bg.wasm` into your application directory. (e.g. `cp hello_gb.wasm <path_to_your_application_folder>`)
- Create js file `main.js` (or whatever you like) with the following content:

```
var ssvm = require('ssvm');
var vm = new ssvm.VM("hello_bg.wasm")
var ret = vm.RunString("say", "world");
console.log(ret);

ret = vm.RunInt("add", 3, 4);
console.log(ret);

ret = vm.RunUint8Array("reverse", Uint8Array.from([1, 2, 3, 4, 5, 6]));
console.log(ret);

ret = vm.RunInt("add", 999, -111);
console.log(ret);

ret = vm.RunUint8Array("reverse", Uint8Array.from([60, 50, 40, 30, 20, 10]));
console.log(ret);
```

## Execute and check results

```
$ node main.js

hello world
7
Uint8Array [ 6, 5, 4, 3, 2, 1 ]
888
Uint8Array [ 10, 20, 30, 40, 50, 60 ]
```
