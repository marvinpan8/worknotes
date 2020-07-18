## 写在前面

可以用于开发 WebAssembly 的语言比较多，笔者之前也尝试过 AssemblyScript、C++、Rust，相对来说，使用 Rust 开发在开发效率和便捷性、包体积大小等方面还是有很大优势的，因此，笔者也建议使用 Rust 来作为 WebAssembly 的开发语言。

Rust 开发 WebAssembly 非常方便，实际上官方周边文档已经比较全面和友好了，而这篇文章主要有两个目的：

- **帮助大家快速上手：**目前有些资料还是比较零散的，**这里希望将各个资料中的一些东西串联起来，**特别是开发者比较关心的入门开发、调试等各个过程。
- **帮助大家建立 Rust 开发 WebAssembly 的心智模型**：由于使用 Rust 入门开发 WebAssembly 已经足够简单，官方实际上把很多内容进行了封装，比如 Rust 和 JS 交互的部分等，**而本文对比较关键的各个部分原理也进行讲解，而不仅仅是如何开发，从而让大家对原理也有一个了解。**

本文的目标读者:

- 对前端有一定经验，并且对 WebAssembly 感兴趣的同学
- 有 Rust 的开发经验，或对使用 Rust 开发 WebAssembly 感兴趣的同学
- 已经使用了 Rust 开发 WebAssembly，尚未深入，但是对整体原理感兴趣的同学

本文看后，读者可以基本掌握：

- 搭建一个简单的 Rust webassembly 开发环境
- 编写代码完成 Rust 和 js 的交互需求，并了解原理
- 调试和错误处理

## Rust+WebAssembly 的能力

在开始开发之前，我们可以先大致了解下 Rust+webassembly 能干些什么：

- 可以使用 Rust std，可以使用 Rust 的大多数第三方库（部分涉及多线程的，可能会有一些问题，关于 webassembly 多线程后续会写文章单独进行讲解）。
- 可以调用几乎任何 JS 侧声明的方法，也可以暴露方法给 JS 调用。
- 可以和 JS 侧互相”传递“几乎任何的数据类型，包括但不限于基本的数字、字符串、对象、Dom对象等。
- 可以直接在 Rust 侧“操作”Dom，甚至已经出现了 Rust 版本的 react

## 起步开发

我们的第一个目标，肯定是希望能最快看到 hello-world，接下来我们需要一步步操作：
安装 [wasm-pack](https://link.zhihu.com/?target=https%3A//rustwasm.github.io/wasm-pack/installer/)，wasm-pack 是将 Rust 打包成 wasm 的命令行工具：

```text
curl https://Rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh 
```

然后我们需要安装 [cargo-generate](https://link.zhihu.com/?target=https%3A//github.com/ashleygwilliams/cargo-generate)，后续在样板代码生成的时候需要：

```text
cargo install cargo-generate 
```

接下来，我们需要建立一个项目，这里我们可以使用 [create-wasm-app](https://link.zhihu.com/?target=https%3A//github.com/rustwasm/create-wasm-app) 这个工具，做前端开发的同学想必对这类 create-xx 应该比较熟悉了。

我们可以直接使用 *npm init* 目录来生成一个样板库，并安装依赖：

```text
npm init wasm-app ./myRust && cd myRust && npm i
```

这个时候我们可以使用 *npm start* 来运行代码并且在浏览器访问了，这个时候，我们应该可以看到一个 alert 弹框。

不过当我们看入口代码发现，这个样板库中的 wasm 部分，是直接引入的一个 npm 包：

```text
import * as wasm from "hello-wasm-pack";
```

这显然是不能符合我们的需求的，因为我们是需要开发 Rust 部分代码的，而不仅仅是前端引入。

这个时候，我们可以借助 [wasm-pack-template](https://link.zhihu.com/?target=https%3A//github.com/rustwasm/wasm-pack-template)，在项目目录下建立一个 Rust wasm 项目文件：

```text
cargo generate --git https://github.com/Rustwasm/wasm-pack-template.git --name hello
```

这样我们所有跟 Rust 有关的 wasm 文件都放在 hello 下面。

我们可以在 hello/src/lib.rs 下面随便修改一点 greet 函数的内容（应该只有一行，随便改），然后运行 *wasm-pack build*

接下来我们修改我们 js 代码的引入：

```text
import * as wasm from "./hello/pkg/hello";
```

接下来我们再运行 *npm start*，应该能看到预期的内容了。

于是，我们已经搭建好了一个方便的 Rust -> wasm 的运行环境，这个环境虽然相对简陋无法直接在实际项目中使用，但对于我们调试和建立起对代码开发的认知，已经足够。

## Rust 和 JS 代码交互

这里的内容是我们在日常开发中使用比较多的，我们的 wasm 模块大多作为 JS 的 enhancment，自然少不了与 JS 的代码交互，这里我们对此进行分析。

## 函数调用

## 暴露函数给 JS 调用

如果是需要暴露在 JS 中调用的函数，我们只需要使用 wasm_bindgen 过程宏即可，一个最简单的例子：

```text
#[wasm_bindgen]
pub fn get_version() -> i32 {
   1
}
```

这个函数经过 wasm-pack 打包之后，可以直接挂到 wasm 模块实例上，当然，我们打包后的代码还会生成一个 js wrapper（所有的 wasm 函数，都会有对应的 js wrapper 函数供调用方使用），最后的返回结果类似如下：

```text
/**
* @returns {number}
*/
export function get_version() {
 var ret = wasm.get_version();
 return ret;
}
```

wasm_bindgen 可以通过传递参数来实现更加复杂的功能，本文章暂不展开，具体可以参考[这里](https://link.zhihu.com/?target=https%3A//rustwasm.github.io/wasm-bindgen/reference/attributes/index.html)。

## 调用 JS 的函数

我们可以在 Rust 层调用 js 几乎任意的函数，只需声明即可，例如调用 js 中的 console.log：

```text
#[wasm_bindgen]
extern {
 #[wasm_bindgen(js_namespace = console)]
 pub fn log(s: &str);
}
```

其原理是，在工具链解析的时候会在 js wrapper 层生成一个对应的函数，然后这个对应的函数会在 wasm 实例化的时候通过 importObject 传递进去（参考[这里](https://link.zhihu.com/?target=https%3A//developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/compileStreaming)的参数传递）。

```text
export const __wbg_log_20c778ed882114c1 = function(arg0, arg1) {
 console.log(getStringFromWasm0(arg0, arg1));
};
```

## Rust 传值给 JS

在了解值传递的过程前，我们需要知道：

- wasm 一个模块只有一个线性内存，这个线性内存是一个在 JS 中分配的 TypedArray，所有的这个模块的相关内容都是存储在这里（原话：(the stack, the heap, everything)）。
- 对于 Rust - wasm 来说，虽然 JS 可以管理这段线性内存，但是为了保证内部的一致性，**所有内存具体分配的操作都是在 Rust 侧完成**，即使 JS 需要写内存，也是调用 Rust 的内存分配函数并传递长度，从 Rust 这里拿到一个偏移量，从而写入。

因此，如果 wasm 需要传递值给 js，也是写入到线性内存的某处，给 JS 读取：

- 如果是简单的数字、字符串，可以直接返回或转成 buffer 后给 JS 读取，一般官方实现了相关 trait，我们直接使用即可。
- 如果是比较复杂的类，需要先序列化成字符串或数组等可序列化的内容（JSON、protobuf等），然后给 JS 调用，具体可以参考下面的使用说明。

如果需要使用 JSON 序列化来返回对象给 JS，我们需要修改我们的 cargo.toml 的相关依赖和 features：

```text
wasm-bindgen = { version = "0.2.58", features = ["serde-serialize"]  }
serde = { version = "1.0.80", features = ["derive"] }
serde_derive = "^1.0.59"
```

然后在代码中调用：

```text
#[derive(Serialize, Deserialize)]
pub struct Dog {
    index: i32
}

#[wasm_bindgen]
pub fn get_dog() -> JsValue {
    let dog = Dog {
        index: 10
    };
    JsValue::from_serde(&dog).unwrap()
}
```

一般情况而言，我们在 Rust 中是没有办法返回 struct 等一些复杂的数据结构给 js 的，不过，我们也可以通过实现相关 trait 来完成返回一个 struct：

```text
pub struct Duck {
    index: i32
}
impl wasm_bindgen::describe::WasmDescribe for Duck {
    fn describe() { 
        u32::describe()
    }
}
impl wasm_bindgen::convert::IntoWasmAbi for Duck {
    type Abi = u32;
    fn into_abi(self) -> u32 {
        self.index as u32
    }
}
#[wasm_bindgen]
pub fn get_version() -> Duck {
   Duck {
       index: 4
   }
}
```

我们可以看出，这样其实还是比较麻烦的，而且效率也不高，所以我们应该尽量减少复杂数据结构的传递。

## Rust 使用 JS 传递的值 && 调用 web 浏览器接口

Rust 使用 JS 传递的值，对于简单类型（数字、字符串）来说，其流程一般是：

- 大部分数字类型，都可以直接传递，也不需要写入线性内存

- 少量的数字类型（比如 i64，实际上对应到 js 是 BigInt），会根据高低位转化成两个数字，也可以认为是直接传递的

- 字符串类型，一般比较复杂，流程分下面几个步骤：

- - 通过 TextEncoder 以 utf-8 的形式编码成 buffer
  - 调用 Rust wasm 提供的的 malloc 函数，拿到一个指针
  - 把之前的 buffer 拷贝到对应的位置

我们可以看到，这种转化特别是字符串的转化，还是比较麻烦的，而实际上我们在一个 wasm 模块中，有的时候并不需要把 js 侧的内容完全拷贝过去，也不会直接使用到 js 的变量，而只是暂时存起来供后面调用，实际上后面也是调用 js 的函数调用，这里流程大概是：

- Rust层先存起来一个JS 对象
- 后面调用 JS 侧函数，仍然JS层调用这个 JS 对象

实际上根本不需要把整个对象放到 Rust 中。

另外有的时候，我们没有办法也不能把一个 js 对象完全传递给 Rust wasm模块中（例如一个 dom 对象），所以，在 Rust wasm 中实际上还有一种 js 变量的“借用”机制， 下面我们来对此进行分析。

我们的 demo 场景是在 Rust 中操作一个 dom 并写入 innerHTML，代码如下：

实际上，getElementById 这些过程在 Rust 侧做也都是可以的，但是这里我们为了突出重点，进行了简化。

```text
// Rust：
#[wasm_bindgen]
pub fn set_dom_inner(dom: HtmlElement) {
    dom.set_inner_html("This is from Rust");
}


// js：
wasm.set_dom_inner(document.getElementById('wasm'));


// html：
<p id="wasm"></div>
```

这个代码中的 Rust 部分，编译出来的 js-wrapper 代码如下：

```text
/**
* @param {any} dom
*/
export function set_dom_inner(dom) {
    wasm.set_dom_inner(addHeapObject(dom));
}
```

这里我们可以看到，其并没有通过一番转化直接把dom“传递进去”（实际上也没法这样做），而是调用了 addHeapObject ：

```text
function addHeapObject(obj) {
 if (heap_next === heap.length) heap.push(heap.length + 1);
 const idx = heap_next;
 heap_next = heap[idx];
 if (typeof(heap_next) !== 'number') throw new Error('corrupt heap');
 heap[idx] = obj;
 return idx;
}
const ret = getObject(arg0).createElement(getStringFromWasm(arg1, arg2));
let index = addHeapObject(ret);
```

这个函数，实际上就是保持住这个对象的引用，防止在 js 侧被垃圾清除，同时传递给 Rust 侧一个索引，在 Rust 层直接存储这个索引即可（ Rust 会生成一个 JsValue 结构体，用来存储这个 u32 的索引）。

当然了，这个时候我们还是有一个问题，既然这个变量被挂到了 heap 上，那肯定也有一个清除机制，否则就是内存泄漏了。

清除机制当然是有的：

```text
function dropObject(idx) {
 if (idx < 36) return;
    heap[idx] = heap_next;
    heap_next = idx;
}
// heap 这里其实是一个链表，把所有为空的串起来了。这样被解除引用了的，会被垃圾回收


function takeObject(idx) {
 const ret = getObject(idx);
    dropObject(idx);
 return ret;
}
export const __wbindgen_object_drop_ref = function(arg0) {
    takeObject(arg0);
};
```

在 wasm 侧，通过调用 import 进来的 __wbindgen_object_drop_ref 最终调用 dropObject 进行清除，__wbindgen_object_drop_ref 的调用是在对应对象在 Rust 析构的时候进行（上面的 dom 对象传递到 rust 后，就是一个 JsValue struct）：

```text
pub struct JsValue {
    idx: u32,
    _marker: marker::PhantomData<*mut u8>, // not at all threadsafe
}
// many other things...
impl Drop for JsValue {
    #[inline]
    fn drop(&mut self) {
        unsafe {
            // We definitely should never drop anything in the stack area
            debug_assert!(self.idx >= JSIDX_OFFSET, "free of stack slot {}", self.idx);
            // Otherwise if we're not dropping one of our reserved values,
            // actually call the intrinsic. See #1054 for eventually removing
            // this branch.
            if self.idx >= JSIDX_RESERVED {
                __wbindgen_object_drop_ref(self.idx);
            }
        }
    }
}
```

> 注：以上这些内容。wasm-pack 工具链都会帮助我们自动完成

## 代码调试与错误处理

比较遗憾的是，目前 WebAssembly 还没有办法直接进行断点调试，也没有办法从 panic! 中恢复（来自官方团队：*in wasm panics always get translated into aborts, so you can't catch them*）。

目前我们能做的事情有：

- 调用 console 相关的方法打印内容到控制台
- 补获 panic!，虽然此时不能恢复了，但是可以打印堆栈到控制台，也可以调用 JS 的函数，上报堆栈和日志等内容
- 可以返回错误给 JS 用于 try catch（推荐做法）

借助以上功能，实际上我们已经可以编写出比较稳妥的 wasm 包了。

## 在 Rust 中使用 console

在 Rust 中使用 console 对象上的方法和使用任何 JS 对象的方法一样，实际上非常简单：

```text
#[wasm_bindgen]
extern {
 #[wasm_bindgen(js_namespace = console)]
 pub fn log(s: &str);
 #[wasm_bindgen(js_namespace = console)]
 pub fn info(s: &str);
 #[wasm_bindgen(js_namespace = console)]
 pub fn warn(s: &str);
 #[wasm_bindgen(js_namespace = console)]
 pub fn error(s: &str);
}
```

直接使用上面的 log 需要传递字符串引用，比较繁琐，我们可以实现一个声明宏来完成这个事情：

```text
macro_rules! log {
    ($($t:tt)*) => (log(&("[W]".to_string() + &format_args!($($t)*).to_string())))
}
```

## 捕获 panic!

为了在 Rust 中捕获 panic，我们需要用到 [console_error_panic_hook](https://link.zhihu.com/?target=https%3A//crates.io/crates/console_error_panic_hook) 这个库，然后我们在某个提供给 JS 的初始化函数中调用 console_error_panic_hook::set_once(); 或者提供给一个单独的函数给 JS 调用。

实际上，console_error_panic_hook 这个函数的代码非常少，在实际项目中，也可以自行修改源码，将项目中需要的 panic 处理通用化。

## 返回错误给 JS 进行 try catch

上面提到我们在 Rust 中虽然能捕获到 panic!，但是此时也只能做“通知”而不能恢复了，而在实际的编码中我们使用的更多的应该是 Result，一个简单的例子如下：

```text
// Rust：
#[wasm_bindgen]
pub fn return_error() -> Result<i32, JsValue>{
 return Err("This is a Js Value".into());
}

// js:
try {
  wasm.return_error();
} catch(e) {
  console.log('catch error:', e);
}
// catch error: This is a Js Value
```

我们还可以借助一些 Rust 错误处理库比如 error_chain 等，来更完善地返回错误。

<完>