# FFI Navigator

[![Build Status](https://dev.azure.com/ffi-navigator/ffi-navigator/_apis/build/status/tqchen.ffi-navigator?branchName=master)](https://dev.azure.com/ffi-navigator/ffi-navigator/_build/latest?definitionId=1&branchName=master)

Most modern IDEs support find function definition within the same language(e.g. python or c++). However, it is very hard to do that for cross-language FFI calls. While solving this general problem can be very technically challenging, we can get around it by building a project-specific analyzer that matches the FFI registration code patterns and recovers the necessary information.

This project is an example of that. Currently, it supports the PackedFunc FFI in the Apache TVM project. It is implemented as a [language server](https://microsoft.github.io/language-server-protocol/) that provides getDefinition function for FFI calls and returns the location of the corresponding C++ API in the TVM project. It complements the IDE tools that support navigation within the same language. We also have preliminary support for MXNet, DGL, and PyTorch, so we can do goto-definition from Python to C++ in these projects too.
 
随着python的流行，需要为了支持效率的python项目都会有和互相调用c++的部分。 
这个问题在像TVM这样的大项目里面都会存在。如何看项目代码就会成为大家都会问的问题。
虽然大部分的IDE都会支持代码跳转功能，可以给定一个函数跳转到函数的定义或者找到函数本身的引用。

这些功能都只存在于一种语言里面（python内部的跳转或者c++内部的跳转），一旦碰到了FFI调用IDE的跳转功能就不好用了。在过去的假期里面，我们开始思考如何帮助TVM社区解决这个问题，然后完成了一个有趣的插件。

如何支持跨语言跳转分析通过一般的程序语言分析去解决跨越FFI跳转本身是一件很难的事情。既然这个一般的问题是比较困难的，我们可以想办法绕过这个问题。因为FFI本身的限制，每个项目一般会有自己的FFI规范。

比如tvm在c++端注册函数的时候会采用**TVM_REGISTER_GLOBAL(“xyz.MyFunc”)**而通过在python端的xyz.py 里面的init_api(“xyz”)来暴露对应的函数。

我们只要支持xyz.MyFunc到对应的c++跳转就可以了。

所以我们的思路是避开程序分析，而直接采用针对每个项目FFI编写规范的模式匹配，去匹配对应的FFI注册函数和ffi调用函数，从而达到支持跳转的目的。

Language Server Protocol有了基本思路，我们可以开始实现这个功能。一开始的时候我们打算直接实现一个vscode plugin。然后社区的小伙伴跑出来说我希望用emacs。是不是可以实现一个统一的插件来支持这个功能呢？经过搜索之后我们发现答案是肯定的。

微软在开发vscode的过程中推出了一个叫language server protocol的规范（https://microsoft.github.io/language-server-protocol/） 大意是我们可以把IDE自动补全跳转等功能实现成一个json rpc server。

然后各个IDE可以直接和这个language server交互从而获得这个功能。看到这里不禁感叹现代的IDE的abstraction decoupling已经好到这个程度。

我们通过实现一个language server，就可以支持所有和LSP交互的IDE了。项目地址具体的效果如下（vscode），目前我们支持tvm项目中从c++和python之间的函数调用跳转以及类型object定义的跳转。除了tvm最近小伙伴还加入了对pytorch，mxnet，dgl的支持，有兴趣的同学也可以尝试一下。希望对大家开发有所帮助

## Installation

Install python package
```bash
pip install --user ffi-navigator
```

### VSCode

See [vscode-extension](vscode-extension)

### Emacs

Install [lsp-mode](https://github.com/emacs-lsp/lsp-mode)

Add the following configuration
```el
(lsp-register-client
 (make-lsp-client
  :new-connection (lsp-stdio-connection '("python3" "-m" "ffi_navigator.langserver"))
  :major-modes '(python-mode c++-mode)
  :server-id 'ffi-navigator
  :add-on? t))
```

- Use commands like `M-x` `lsp-find-definition` and `M-x` `lsp-find-references`

If you use eglot instead, check out [this PR](https://github.com/tqchen/ffi-navigator/pull/1).
eglot does not support multiple servers per language at the moment, the PR above contains a workaround.

### Other editors/IDEs

It should be straightforward to add support for other editors or IDEs that have a LSP client implementation.
Please refer to this [site](https://langserver.org/) for the availability of clients.

Since ffi-navigator is intended to be used with other general purpose servers such as [pyls](https://github.com/palantir/python-language-server), your LSP client needs to be able to talk to multiple servers per language. If your client does not have such feature, check out [this PR](https://github.com/tqchen/ffi-navigator/pull/1) and the discussion there for a workaround. This [branch](https://github.com/masahi/tvm-ffi-navigator/tree/pyls-fallback) has fallback to pyls and it is always kept up to date with the master branch.


## Features

### TVM FFI

- Find definition/references of FFI objects(e.g. PackedFunc in TVM) on the python and c++ side.
  - Jump from a python PackedFunc into ```TVM_REGISTER_GLOBAL```, ```@register_func```
- Find definition/references of FFI objects on the python and c++ side.
  - move cursor to the object class name on the python side.
  - move cursor to the ```_type_key``` string on the c++ side.

### PyTorch

- Jump to C10 registered ops. In python they corresponds to functions under `torch.ops` namespace.
  - Example: `torch.ops.quantized.conv2d (py)` -> `c10::RegisterOperators().op("quantized::conv2d", ...) (cpp)`
- Jump to cpp functions wrapped by pybind.
  - Example: `torch._C._jit_script_class_compile (py)` -> `m.def( "_jit_script_class_compile", ...) (cpp)`


## Development

For developing the python package locally, we can just make sure ffi_navigator is in your python path in bashrc.
```bash
export PYTHONPATH=${PYTHONPATH}:/path/to/ffi-navigator/python
```

### Project Structure

- python/ffi_navigator The analysis code and language server
- python/ffi_navigator/dialect Per project dialects
- vscode-extension language server extension for vscode

### Adding Support for New FFI Patterns

Add your FFI convention to [dialect namespace](python/ffi_navigator/dialect).


## Demo

### VSCode

See [vscode-extension](vscode-extension)

### Emacs

#### Goto definition from Python to C++
![goto-def-py-cpp](https://github.com/tvmai/web-data/blob/master/images/ffi-navigator/emacs/tvm_find_def_py_cpp.gif)
#### Goto definition from C++ to Python
![goto-def-py-cpp](https://github.com/tvmai/web-data/blob/master/images/ffi-navigator/emacs/tvm_find_def_cpp_py.gif)
#### Find reference across Python and C++
![goto-def-py-cpp](https://github.com/tvmai/web-data/blob/master/images/ffi-navigator/emacs/tvm_find_reference.gif)
#### Goto definition in PyTorch
![goto-def-py-cpp](https://github.com/tvmai/web-data/blob/master/images/ffi-navigator/emacs/torch.gif)
