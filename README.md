# KlyScanner

![License](https://img.shields.io/badge/License-BSL%201.0-blue) ![C++20](https://img.shields.io/badge/C%2B%2B20-Standard-purple)

**A background console command-input library for C++, powered by replxx and Brigadier, with KlyLogger integration.**

**一个基于 replxx 与 Brigadier 的 C++ 后台控制台命令输入库，支持与 KlyLogger 集成。**

---

## Platforms / 平台支持

- Windows: MSVC or clang-cl; GCC/MinGW is not supported by this project's CMake configuration.
- Linux: GCC or Clang, subject to the requirements of the dependencies.
- Requires C++20, CMake 3.20 or newer, and a toolchain with IPO/LTO support.

Windows 构建支持 MSVC 与 clang-cl，不支持 GCC/MinGW。Linux 可使用 GCC 或 Clang，并需满足各依赖库的要求。
项目要求 C++20、CMake 3.20 及以上版本，以及支持 IPO/LTO 的工具链。

The current regression checks have been run with MSVC on Windows. Interactive rendering depends on the terminal and should be verified in the target environment.

当前回归检查已在 Windows/MSVC 环境运行；交互显示效果取决于终端，请在目标环境验证。

---

## Usage / 使用示例

Configure commands and callbacks first, then start input. All public methods are static; no `KlyScanner` instance is needed.

先配置命令与回调，再启动输入。所有公开接口均为静态方法，无需创建 `KlyScanner` 实例。

```cpp
#include "KlyScanner.hpp"
#include <cstdlib>

int main() {
	KlyScanner::set_syntax_exception_callback([](const std::wstring &message) {
		KlyLogger().error(message);
	});
	KlyScanner::set_exception_callback([](const std::wstring &type, const std::wstring &message) {
		KlyLogger().error(L"{}: {}", type, message);
	});

	KlyScanner::register_command(L"hello").Executes([](auto &) {
		KlyLogger().info(L"Hello, world!");
		return 1;
	});

	// Register explicitly; no exit command is installed by the library.
	// 显式注册退出命令；库本身不提供内置 exit 命令。
	KlyScanner::register_command(L"exit").Executes([](auto &) {
		KlyLogger::wait();
		std::exit(0);
		return 0;
	});

	KlyScanner::start(); // Background input; returns without waiting / 后台输入，不等待线程结束
	KlyLogger().info(L"Try hello or exit.");
	KlyScanner::join();  // Wait for the input worker / 等待输入线程结束
	KlyLogger::wait();
	return 0;
}
```

The prompt is `> `. Tab completion, inline hints, syntax highlighting, and in-memory input history are provided through replxx.
Narrow terminal text is converted from UTF-8 to `std::wstring` through `KlyLogger::StringConverter` before command execution.

提示符为 `> `。replxx 提供 Tab 补全、行内提示、语法高亮与内存中的输入历史。
终端的 UTF-8 文本通过 `KlyLogger::StringConverter` 转为 `std::wstring` 后交给命令分发器。

### Command arguments / 命令参数

`register_command()` returns a Brigadier literal builder. Continue building the command with `Then` and `Executes`.
The dispatcher source type is fixed to `void *`; Scanner supplies `nullptr` when parsing or executing command text.

`register_command()` 返回 Brigadier 字面量构建器，可继续通过 `Then`、`Executes` 构建命令。
分发器的 source 类型固定为 `void *`；Scanner 解析或执行命令文本时传入 `nullptr`。

```cpp
KlyScanner::register_command(L"square")
	.Then<brigadier::Argument, brigadier::Integer>(L"value", 0, 100)
	.Executes([](auto &context) {
		const auto value = context.template GetArgument<brigadier::Integer>(L"value");
		KlyLogger().info(L"{} squared is {}", value, value * value);
		return 1;
	});
```

Command handlers and Brigadier consumers currently use function pointers: use named functions or non-capturing lambdas.
Scanner's callback setters accept `std::function` and also support capturing lambdas.
See [brigadier-cpp](brigadier-cpp/README.md) for the builder and argument APIs.

命令执行函数与 Brigadier consumer 当前使用函数指针，请使用普通函数或无捕获 lambda。
Scanner 的回调 setter 接受 `std::function`，可以使用带捕获的 lambda。
构建器及参数接口详见 [brigadier-cpp](brigadier-cpp/README.md)。

---

## API Reference / 接口参考

The public API is declared in [`KlyScanner.hpp`](KlyScanner.hpp).
Dispatcher wrappers operate directly on the shared command tree; they do not enqueue work on the input thread.

公开接口声明于 [`KlyScanner.hpp`](KlyScanner.hpp)。分发器封装直接访问共享命令树，不会把操作排入输入线程。

### Starting and waiting / 启动与等待

```cpp
KlyScanner::start(); // Start once without waiting / 只启动一次，不等待结束
KlyScanner::join();  // Start if needed, then wait / 尚未启动时先启动，再等待
```

- Registering commands or callbacks does not start input implicitly.
- Multiple `start()` calls do not create additional workers. Concurrent or repeated `join()` calls share one underlying thread join.
- Calling `join()` from the input worker returns immediately instead of waiting for itself.
- EOF ends the worker; Ctrl-C aborts the current input line and resumes prompting. The worker cannot be restarted after it finishes.
- There is no explicit `stop()` API. `join()` waits; it does not cancel a pending input read.

注册命令或回调不会隐式启动输入。重复 `start()` 不会创建多个线程；多个线程同时或重复调用 `join()` 时，只进行一次底层 join。
输入线程自己调用 `join()` 会立即返回。EOF 结束线程，Ctrl-C 取消当前输入行后继续提示；线程结束后不能重新启动。
当前没有显式 `stop()` 接口，`join()` 只等待，不会取消正在阻塞的输入读取。

### Console output / 控制台输出

Prefer KlyLogger for normal application output, including status messages, diagnostics, warnings and errors.
Reserve `KlyScanner::print()` for output that should not become a log record, such as interactive questions,
selection instructions or input hints.
`print()` only writes text: it does not read a response, change Scanner's `> ` prompt, or wait for input.

常规输出建议使用 KlyLogger，包括运行状态、诊断信息、警告和错误。
`KlyScanner::print()` 仅用于不走日志的输出，例如交互提问、选项说明、输入提示。
`print()` 只负责输出，不会读取回答、修改 Scanner 的 `> ` 提示符，也不会等待用户输入。

```cpp
KlyLogger().info(L"应用已启动"); // Normal output goes through logging / 常规输出使用日志
KlyScanner::print(L"请选择操作：输入 help 查看命令。\n"); // Non-log UI text / 非日志交互文字

// printf-style formatting, not fmt's {} syntax / printf 风格格式，不是 fmt 的 {} 语法
KlyScanner::print("name=%s, count=%d, value=%.2f\n", "test", 3, 1.25);

// Wide text: exactly one argument, converted through KlyLogger / 宽字符串仅接受一个参数，由 KlyLogger 转码
KlyScanner::print(L"中文输出，进度 100%\n");
std::wstring message = L"任务完成\n";
KlyScanner::print(message);

// Pass arbitrary narrow text as data, not as the format / 任意窄字符串应作为数据，而非格式串
std::string text = "progress: 100%\n";
KlyScanner::print("%s", text.c_str());
```

`print(const char *format, ...)` uses the same `printf` formatting as replxx; argument types must match the format specifiers.
Pass C strings (for example, `value.c_str()`) to `%s`, not `std::string` objects.
`print(const std::wstring &text)` accepts one argument only: it converts the text to UTF-8 with `KlyLogger::StringConverter::to_string()`
and calls the narrow overload with `"%s"`, so percent signs are printed literally.

`print(const char *format, ...)` 使用与 replxx 相同的 printf 格式化规则，参数类型必须匹配格式符。
`%s` 应传入 C 字符串（如 `value.c_str()`），不能直接传入 `std::string` 对象。
`print(const std::wstring &text)` 仅接受一个参数，使用 `KlyLogger::StringConverter::to_string()` 转为 UTF-8，
再以 `"%s"` 调用窄字符串重载，因此宽字符串中的百分号按原文输出。

Both overloads use replxx's output path to cooperate with the input prompt and handle ANSI escape sequences.
They do not append a newline, start the input worker, or emit KlyLogger records or Scanner execution callbacks.
If the narrow format pointer is null or formatting fails the internal checks, `print()` returns without printing.

两个重载均通过 replxx 输出，以配合输入提示并处理 ANSI 转义序列；不会自动换行、启动输入线程、产生日志记录或触发 Scanner 的执行回调。
窄字符串格式指针为空，或格式化未通过内部检查时，`print()` 直接返回，不执行输出。

### Callbacks / 回调

```cpp
KlyScanner::set_before_execute_callback([] {
	// Before a non-empty entered command / 非空输入命令执行前
});
KlyScanner::set_after_execute_callback([] {
	// After execution, including command exceptions / 执行后，包括命令抛异常的情况
});
KlyScanner::set_syntax_exception_callback([](const std::wstring &message) {
	// CommandSyntaxException::What()
});
KlyScanner::set_exception_callback([](const std::wstring &type, const std::wstring &message) {
	// typeid(exception).name() and what(), converted to wide strings
	// 转为宽字符串后的异常类型名称与 what() 消息
});
KlyScanner::set_after_log_callback([](const std::wstring &original, const std::wstring &plain) {
	// Runs on KlyLogger's worker, not the input worker / 在日志线程执行，而非输入线程
});

KlyScanner::set_before_execute_callback({}); // Clear a callback / 清除回调
```

Execution and exception callbacks run on the input worker.
The execution order is `before -> command -> after`; command exceptions are then forwarded to the appropriate exception callback.
Execution-hook exceptions are ignored so that they do not prevent execution or replace the command's exception.
If a command terminates the process with `std::exit`, the after-execute callback does not run.

执行及异常回调在输入线程运行。顺序为 `before -> command -> after`，命令异常随后转发到对应异常回调。
执行前后回调自身的异常会被忽略，不阻止命令执行，也不替换命令异常。命令调用 `std::exit` 后不会返回，因此不触发执行后回调。

The syntax callback receives `What()`.
The standard-exception callback receives the unmodified `typeid(exception).name()` and `exception.what()`, both converted using `KlyLogger::StringConverter`.
Type names are compiler-dependent and are not demangled. With no callback installed, the corresponding error is not logged automatically.

语法异常回调接收 `What()`；标准异常回调接收通过 `KlyLogger::StringConverter` 转换后的原始类型名和 `what()`。
类型名依赖编译器，库不进行解码或裁剪；未设置回调时，不会自动输出对应错误日志。

Callback registration is synchronized; invocation uses a snapshot outside the lock. An already captured callback may still run after replacement.
Keep captured objects alive until no callback can use them. Avoid logging from the after-log callback itself, which can produce an endless stream of callbacks.

回调注册有锁保护，调用时使用锁外快照；替换回调后，已取得的旧快照仍可能被执行。
请保证捕获对象的生命周期覆盖回调使用期间，不要在日志后回调中再次输出日志，以免不断触发新的回调。

### Direct execution and parsing / 直接执行与解析

```cpp
int result = KlyScanner::execute_command(L"hello");

brigadier::StringReader reader(L"hello");
result = KlyScanner::execute_command(reader);

auto parsed = KlyScanner::parse_command(L"hello");
result = KlyScanner::execute_command(parsed);

brigadier::StringReader anotherReader(L"hello");
auto anotherParse = KlyScanner::parse_command(anotherReader);
```

These methods run on the calling thread. They do not trigger Scanner's execution hooks or exception callbacks;
exceptions propagate to the caller. Executing a `ParseResults<void *>` uses the source already stored in the parse result.

这些方法在调用线程执行，不触发 Scanner 的执行前后及异常回调；异常直接交给调用方。
执行 `ParseResults<void *>` 时，使用解析结果中已有的 source。

### Command tree and suggestions / 命令树与补全

| Method / 方法 | Behavior / 行为 |
|---|---|
| `register_command(name)` | Register or retrieve a top-level literal builder / 注册或获取顶层字面量构建器 |
| `get_root_command_node()` | Return the root by reference without copying it / 返回根节点引用，不复制节点 |
| `find_node(path)` | Return an optional borrowed node reference / 返回可选的借用节点引用 |
| `get_node_path(target)` | Return a node's path as `vector<wstring>` / 获取节点路径 |
| `get_all_command_usages(node, restricted)` | Build usage strings; source is `nullptr` / 获取用法列表，source 为 `nullptr` |
| `get_smart_command_usage(node)` | Build compact usages keyed by node pointers / 获取以节点指针为键的简洁用法 |
| `find_ambiguities_commands(consumer)` | Report ambiguous sibling nodes / 扫描并报告节点歧义 |
| `get_completion_suggestions(parse, cancel)` | Return a future for suggestions at the input end / 返回输入末尾补全结果的 future |
| `get_completion_suggestions(parse, cursor, cancel)` | Request suggestions at a wide-string cursor offset / 获取指定宽字符串偏移处的建议 |
| `set_brigadier_consumer(consumer)` | Set the per-command execution result consumer / 设置命令执行结果 consumer |

```cpp
auto &root = KlyScanner::get_root_command_node(); // Keep the reference / 保留引用
auto usages = KlyScanner::get_all_command_usages(&root, false);

if (auto node = KlyScanner::find_node({L"hello"})) {
	auto &value = node->get(); // optional<reference_wrapper<CommandNode<void *>>>
	auto path = KlyScanner::get_node_path(&value);
}

auto parsed = KlyScanner::parse_command(L"he");
auto suggestions = KlyScanner::get_completion_suggestions(parsed).get();
for (const auto &suggestion : suggestions.GetList()) {
	// Keep suggestions alive while using its list / 遍历期间保持 suggestions 存活
	KlyLogger().info(suggestion.GetText());
}

KlyScanner::set_brigadier_consumer([](brigadier::CommandContext<void *> &, bool success, int result) {
	// Receives execution results, including direct execute_command calls
	// 接收执行结果，也适用于直接 execute_command 调用
});
```

`find_node()` returns `std::nullopt` for a missing path. References and node pointers do not own nodes;
do not retain them after the corresponding nodes have been removed or replaced.
Keep parse results and any cancellation storage alive until their suggestion future completes.
The optional `bool *cancel` is forwarded to Brigadier; it is not a thread-safe cancellation token.

路径不存在时，`find_node()` 返回 `std::nullopt`。引用和节点指针不拥有节点，节点被删除或替换后不要继续使用。
补全 future 完成前，必须保持解析结果及可选取消标志存活。`bool *cancel` 原样传给 Brigadier，并不是线程安全的取消令牌。

---

## Threading and logging / 线程与日志

Configure the command tree before `start()` or `join()`. Dispatcher wrappers and returned builders are not synchronized;
do not mutate commands while the input worker or another thread is parsing, completing, or executing against the tree.
Callback setters are synchronized, but user callback state still needs its own synchronization when shared across threads.

请在 `start()` 或 `join()` 前完成命令树配置。分发器封装与返回的构建器没有并发保护；输入线程或其他线程正在解析、补全、执行时，不要修改命令树。
回调 setter 有同步保护，但用户回调里跨线程共享的状态仍需自行同步。

Starting Scanner installs KlyLogger's global after-log callback to integrate logging with the prompt.
Use `KlyScanner::set_after_log_callback()` for an additional user hook. A pre-existing global after-log callback is replaced;
setting KlyLogger's global callback again later replaces Scanner's integration.

启动 Scanner 会安装 KlyLogger 的全局日志后回调，用于日志与输入提示的集成。
额外的用户回调请通过 `KlyScanner::set_after_log_callback()` 设置。原有全局日志后回调会被覆盖；之后再次设置 KlyLogger 的全局回调，也会覆盖 Scanner 的集成。

The internal runtime is intentionally retained for the process lifetime rather than destroyed during static cleanup.
This avoids destroying an active Scanner thread or joining it from itself when a command calls `std::exit`.
It is not an orderly Scanner shutdown, does not wait for arbitrary user work, and cannot make other libraries' exit handlers thread-safe.
`std::exit` does not unwind local objects. Flush required output explicitly, and do not assume forceful process termination will flush logs.

内部 runtime 有意保留到进程结束，不参与静态对象清理，避免命令调用 `std::exit` 时销毁活跃线程或让线程 join 自己。
这不是有序停止 Scanner 的流程，不等待任意用户任务，也不能保证其他库的退出处理线程安全。
`std::exit` 不展开局部对象；需要保留的输出应显式等待完成，强制终止进程不保证日志完整。

---

## Build / 构建

Clone recursively, including KlyLogger's fmt dependency / 递归克隆，包含 KlyLogger 的 fmt 依赖：

```sh
git clone --recurse-submodules https://github.com/KinnerFisch/KlyScanner.git
cd KlyScanner
# For an existing clone / 已有工作区：
git submodule update --init --recursive
```

### Windows

Use an x64 Developer Command Prompt for MSVC builds on Windows / Windows 下使用 x64 开发者命令提示符：

```sh
cmake -S . -B build
cmake --build build --config Release --target KlyScanner
```

The configuration builds static libraries in Release and requires IPO/LTO. Windows builds produce both `KlyScanner_MD.lib` and `KlyScanner_MT.lib` by default.
Each archive contains Scanner, Brigadier and static replxx objects, all compiled with its matching CRT. KlyLogger (including fmt) remains a separate matching library.
Use a separate build directory when changing compiler toolchains.
Use a linker compatible with the compiler's LTO output; MSVC and clang-cl LTO objects are not interchangeable.

配置生成 Release 静态库并要求 IPO/LTO。Windows 下默认同时生成 `KlyScanner_MD.lib` 和 `KlyScanner_MT.lib`，分别使用对应的 CRT。
每个库均包含 Scanner、Brigadier 和静态 replxx 的对象文件；KlyLogger（已包含 fmt）仍需单独链接匹配版本。
切换工具链时应使用独立构建目录；链接器需匹配编译器的 LTO 产物，MSVC 与 clang-cl 的 LTO 对象不能混用。

To build only one CRT variant / 仅构建一个 CRT 版本：

```sh
cmake --build build --config Release --target KlyScanner_MD
# Or / 或：
cmake --build build --config Release --target KlyScanner_MT
```

### Linux

```sh
cmake -S . -B build
cmake --build build --target KlyScanner --parallel
```

Linux has no MSVC MD/MT distinction. It produces `libKlyScanner.a`, containing Scanner, Brigadier and static replxx;
link it with `libKlyLogger.a`, which already includes fmt. CMake targets also propagate the platform's thread-linking requirements.
A static Scanner archive does not make the entire executable or its C/C++ runtime statically linked.

Linux 没有 MSVC 的 MD/MT 区分，只生成包含 Scanner、Brigadier 和静态 replxx 的 `libKlyScanner.a`；
配合 `libKlyLogger.a` 链接，后者已包含 fmt。CMake 目标还会传递平台所需的线程链接设置。
Scanner 是静态库不代表最终程序或 C/C++ 运行库也完全静态链接。

### CMake integration / CMake 集成

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyConsole LANGUAGES C CXX)

add_subdirectory(path/to/KlyScanner)
add_executable(MyConsole main.cpp)
target_link_libraries(MyConsole PRIVATE KlyScanner::KlyScanner)

if(MSVC)
	set_property(TARGET MyConsole PROPERTY MSVC_RUNTIME_LIBRARY MultiThreadedDLL) # MD
endif()
```

For MT, link `KlyScanner::KlyScanner_MT` and use `MultiThreaded` for the application.
The compatibility alias `KlyScanner::KlyScanner` defaults to MD; `KLY_SCANNER_MSVC_RUNTIME=MT` changes only this alias, not which archives are built.
CMake carries the include paths and library dependencies. When linking manually, include KlyScanner, Brigadier, replxx, KlyLogger and fmt headers;
link `KlyScanner_MD.lib` + `KlyLogger_MD.lib` (or both MT variants). No separate Brigadier/replxx library or replxx DLL is needed.
Use `/NODEFAULTLIB:brigadier.lib` to suppress Brigadier's header auto-link directive, and define `REPLXX_STATIC` when including replxx headers.
The CMake targets propagate both settings automatically. Non-MSVC builds produce a single `libKlyScanner.a` with the same embedded components.
The current install rules do not provide a complete `find_package(KlyScanner)` package; source-tree integration is recommended.

使用 MT 时，链接 `KlyScanner::KlyScanner_MT`，并为应用指定 `MultiThreaded`。
兼容别名 `KlyScanner::KlyScanner` 默认选择 MD；`KLY_SCANNER_MSVC_RUNTIME=MT` 仅切换该别名，不影响同时构建两个版本。
CMake 目标会传递头文件路径与库依赖。手动集成时需提供上述依赖的头文件，链接 `KlyScanner_MD.lib` 与 `KlyLogger_MD.lib`（或均使用 MT）；
无需额外链接 Brigadier、replxx 或分发 replxx DLL。手动链接需设置 `/NODEFAULTLIB:brigadier.lib`，包含 replxx 头文件时定义 `REPLXX_STATIC`，CMake 目标已自动传递两者。
非 MSVC 构建生成包含相同组件的单个 `libKlyScanner.a`。当前安装规则不提供完整的 `find_package(KlyScanner)` 包，建议通过源码目录集成。

---

## Key Features / 主要功能

- Background console input with explicit waiting / 后台控制台输入与显式等待
- Brigadier command tree and fluent builders / Brigadier 命令树与链式构建器
- Tab completion, inline hints, and syntax highlighting / Tab 补全、行内提示和语法高亮
- In-memory command history / 内存中的命令历史
- printf-style console output and a wide-string overload / printf 风格控制台输出与宽字符串重载
- Execution and exception callbacks / 执行前后与异常回调
- KlyLogger integration and UTF-8 conversion / KlyLogger 日志集成与 UTF-8 转换
- Borrowed command-node references without node copies / 借用节点引用，不复制节点

## Licenses and third-party software / 许可证与第三方软件

KlyScanner is distributed under the [Boost Software License 1.0](LICENSE), copyright © 2026 KinnerFisch.
Dependencies retain their own licenses:

KlyScanner 使用 [Boost Software License 1.0](LICENSE) 发布，版权所有 © 2026 KinnerFisch。依赖库保留各自许可证：

- [KlyLogger](KlyLogger/LICENSE): Boost Software License 1.0.
- [brigadier-cpp](brigadier-cpp/LICENSE): MIT License.
- [replxx](replxx/LICENSE.md): BSD-style license; see the full license text.
- [{fmt}](KlyLogger/fmt/LICENSE): MIT License, used by KlyLogger.

## Inspiration / 灵感来源

The interactive console is modeled after PaperConsole-style command input, combining replxx's line editor with Brigadier's command parsing.

交互控制台参考 PaperConsole 风格，将 replxx 的行编辑能力与 Brigadier 的命令解析结合使用。
