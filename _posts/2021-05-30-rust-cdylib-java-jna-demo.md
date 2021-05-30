---
layout: post
title: "java 调用 rust，demo"
tags: jna, java, rust, ffi, cdylib
---

# Foreign Function Interface
FFI是一组机制，目标是让编程语言A写的程序可以使用另一个编程语言B写的程序。比如，在java程序中调用rust程序。

# 具体方法：
目前我只知道一种，就是使用动态链接库,Dynamic-link Library, DLL，Windows上面是 `.dll`文件, Linux上是`.so`文件。
假如需要在java中调用rust代码，就先将要用的rust代码编译成动态库，然后在java程序中调用动态库。rust代码中没有调用者的信息，不关心是谁来调用，java代码中只有动态库的信息，没有产生动态库的源文件的信息。

# JNA
Java 解析dll的机制，[Java Native Access platform](https://github.com/java-native-access/jna)。用起来和c++调用dll差不多，c++ 和rust 里面是`extern "C"`然后写函数声明，编译时链接，java里面就是定义一个对象，函数声明写在对象里，运行时构造对象。

[这里](https://www.eshayne.com/jnaex/index.html)有JNA的使用示例，包括生成库的C代码和调用库的java代码，非常有用。

# rust ffi
用起来和c++挺像的，c++ 里面用 `extern "C"`修饰要导出的函数，rust里面也是 `extern "C"`,不过还要加一个`#[no_mangle]`。

[Rust FFI Omnibus](http://jakegoulding.com/rust-ffi-omnibus/)是一组其他语言调用rust代码的例子，包括C,Ruby,Python,Haskell,Node.js,C#,Julia（没有java）。

# demo
以常用简单类型里面最复杂的字符串为例。C语言中，字符串是表示为字符数组的，FFI中，在两个语言中传递字符串时，实际上传递的是字符数组的指针，动态库负责管理内存，所以被调用的代码需要负责清理它所产生的字符串所占用的内存。

demo中有三个方法：
 - char_cnt，输入字符串，输出字符串中的字符数量（utf-8编码）
 - and_rust，输入字符串，输出原来的字符串末尾加上 `and rust` 的新字符串
 - free_str，输入字符串，释放内存

## java部分
外部方法定义：

```java
package yang.da.learn.java.jna;

import com.sun.jna.Library;

public interface CLib extends Library{
    public int char_cnt(String str);
    public void free_str(String str);
    public String and_rust(String str);
}
```
调用：
```java
package yang.da.learn.java.jna;

import com.sun.jna.Native;

public class Demo {
    public static void main(String[] args) {

        final CLib lib = (CLib) Native.loadLibrary("cdylib", CLib.class);

        // char_cnt: fn(*const c_char) -> isize
        int char_cnt = lib.char_cnt("hello from java");
        System.out.println("hello from java, size: " + char_cnt);

        // and_rust: fn(*mut c_char) -> *mut c_char
        String and_rust = lib.and_rust("hello from java "); // hello from java and rust
        System.out.println(and_rust);

        // free memory
        lib.free_str(and_rust);
    }
}
```
输出：
```
hello from java, size: 15
hello from java and rust
```

java package file tree:
```
|-- build.gradle.kts
`-- src
    `-- main
        |-- java
        |   `-- yang
        |       `-- da
        |           `-- learn
        |               `-- java
        |                   `-- jna
        |                       |-- CLib.java
        |                       `-- Demo.java
        `-- resources
            `-- linux-x86-64
                `-- libcdylib.so

```
有两个细节，`Native.loadLibrary`方法会从系统对应的文件夹里面找库文件，我用的是linux，所以它在`linux-x86-64`文件里找；另外一个细节是`Native.loadLibrary`方法输出的参数并不是库文件的全名，因为不同系统中名字不一样，对于linux系统，需要去掉`lib`前缀和`.so`后缀。

## rust部分
`lib.rs`
```rust
extern crate libc;

use libc::c_char;
use std::ffi::{CString, CStr};

#[no_mangle]
pub extern "C" fn char_cnt(s: *const c_char) -> usize {
    let c_str = unsafe {
        assert!(!s.is_null());

        CStr::from_ptr(s)
    };
    let r_str = c_str.to_str().unwrap();
    r_str.chars().count()
}

#[no_mangle]
pub extern "C" fn free_str(s: *mut c_char) {
    unsafe {
        if s.is_null() {
            return;
        }
        CString::from_raw(s)
    };
}

#[no_mangle]
pub extern "C" fn and_rust(s: *mut c_char) -> *mut c_char {
    let c_str = unsafe {
        assert!(!s.is_null());

        CStr::from_ptr(s)
    };

    let mut r_string = c_str.to_str().expect("not utf-8").to_owned();
    r_string.push_str("and rust");
    let c_string = CString::new(r_string).expect("nul error");
    c_string.into_raw()
}
```
rust中用了`*mut char`来表示字符串，而不是`&str`，我理解是有编码的问题，rust里面的字符串一定是UTF-8。FFI相关的两个字符串类型：`CStr`和`CString`，关系类似`str`和`String`，`CString`是有所有权的，`CStr`没有，释放空间的代码里，`CString::from_raw(s)`就是将一个字符串的内存管理接管过来，由rust来释放。具体参见[这个StackOverflow回答](https://stackoverflow.com/a/24148033)。

Cargo.toml:
```toml
...
[dependencies]
libc = "*"

[lib]
crate-type = ["cdylib"]
```
rust package file tree:
```
|-- Cargo.lock
|-- Cargo.toml
`-- src
    `-- lib.rs
```
## 配合
目前是手动的，`cargo build`命令生成的文件手动复制到resources文件夹中，应该有自动的，目前不知道，如果真的生产中要用，还是需要自动化一下，包括java代码中的函数签名，应该也要能够随着rust代码的变化自动变化，不知道目前有没有这种工具。