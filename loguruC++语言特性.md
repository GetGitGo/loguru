这两个文件以 **C++11** 为上限。`loguru.hpp` 放接口、宏和模板，`loguru.cpp` 放实现。没有 `constexpr`、`static_assert`、`enum class`、`if constexpr`、折叠表达式、结构化绑定，也没有智能指针。

## C++11 语言特性

| 特性                               | 位置                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `nullptr`                          | `loguru.hpp` 279、666、713；`loguru.cpp` 206–216             |
| `auto`                             | `loguru.hpp` 290、895；`loguru.cpp` 534、941、1616           |
| 类型别名 `using`                   | `loguru.hpp` 318 `using Verbosity = int`，888 `using Printer`，982；`loguru.cpp` 176–179、1755 |
| 带底层类型的 `enum`                | `loguru.hpp` 326 `enum NamedVerbosity : Verbosity`。这不是 `enum class`，枚举项在 `loguru` 命名空间里 |
| 右值引用、移动构造                 | `loguru.hpp` 276 `Text(Text&&)`，689 `LogScopeRAII(LogScopeRAII&&)`，890 `Printer&&` |
| `= delete`                         | `loguru.hpp` 281–283、708–710、868–871                       |
| `= default`                        | `loguru.hpp` 704 默认移动构造                                |
| 非静态成员默认初始化               | `loguru.hpp` 418–439、712–717、881                           |
| 花括号初始化                       | `loguru.hpp` 501 `options = {}`，717 `_name[...] = {}`；`loguru.cpp` 211 `{ 0 }`，1173 `REPLACE_LIST = { ... }` |
| 变参模板                           | `loguru.hpp` 304、636、668、725                              |
| 函数模板显式特化 `template<>`      | `loguru.hpp` 743–760 `format_value`                          |
| 类模板偏特化、数组引用             | `loguru.hpp` 930–931 `const char(&)[N]`                      |
| `decltype`                         | `loguru.hpp` 970；`loguru.cpp` 159                           |
| `override`                         | `loguru.hpp` 893                                             |
| `noexcept(false)`                  | `loguru.hpp` 1260、1287；定义在 `loguru.cpp` 1721、1727。析构被允许抛异常 |
| `thread_local`                     | `loguru.cpp` 1061、1761                                      |
| lambda                             | `loguru.cpp` 220 立即调用，1489 作为线程入口                 |
| 范围 `for`                         | `loguru.cpp` 567、941、1028、1187 `auto&&`、1573、1819       |
| 原始字符串 `R"(...)"`              | `loguru.cpp` 1208、1211                                      |
| 列表初始化 `std::initializer_list` | `loguru.cpp` 1173                                            |

移动和删除特殊成员集中在 `Text`：

```276:283:loguru.hpp
		Text(Text&& t)
		{
			_str = t._str;
			t._str = nullptr;
		}
		Text(Text& t) = delete;
		Text& operator=(Text& t) = delete;
		void operator=(Text&& t) = delete;
```

## C++98 仍在大量使用的机制

| 特性                                  | 位置                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| 命名空间、匿名命名空间                | `loguru.hpp` 202–207、268、1067                              |
| `class` / `struct`、访问控制          | `Text` 271，`Message` 360，`LogScopeRAII` 663，`EcEntryBase` 863 |
| 继承、虚析构、纯虚函数                | `loguru.hpp` 867、873、885 `EcEntryData : public EcEntryBase` |
| `explicit`、构造初始化列表            | `loguru.hpp` 274、890、1259                                  |
| RAII                                  | `Text`、`LogScopeRAII`、`EcEntryBase`、`StreamLogger`；`loguru.cpp` 954 `lock_guard` |
| `const` 成员函数                      | `loguru.hpp` 285–286、875                                    |
| 函数重载                              | `loguru.hpp` 311 与 316 的 `textprintf`；1263 与 1270 的 `operator<<` |
| 运算符重载                            | `loguru.hpp` 1263 `operator<<`，返回 `*this` 形成链式调用    |
| 函数指针 `typedef`                    | `loguru.hpp` 399–411                                         |
| 默认参数                              | `loguru.hpp` 501、590–591、787                               |
| `static` 成员函数、`static` 局部/全局 | `loguru.hpp` 441 `SignalOptions::none()`；`loguru.cpp` 200–218 |
| `extern` 全局                         | `loguru.hpp` 378 起一组 `g_*`，定义在 `loguru.cpp`           |
| `inline`                              | `loguru.hpp` 743；`loguru.cpp` 294                           |
| 前向声明                              | `loguru.hpp` 857 `struct StringStream`，定义在 `loguru.cpp` 1742 |
| 引用参数                              | 贯穿 `const Message&`、`const T&`                            |
| `static_cast` / `reinterpret_cast`    | `loguru.hpp` 973；`loguru.cpp` 294、335、458、513            |
| `new` / `delete`                      | `loguru.cpp` 335、687、837、1489、1788                       |
| `try` / `catch`，含 `catch (...)`     | `loguru.cpp` 486、1207、2022                                 |
| `typeid`（RTTI）                      | `loguru.cpp` 1168；头文件 186–199 用编译器宏决定是否开启     |
| C 风格变参 `...`、`va_list`           | `loguru.hpp` 311、651；`loguru.cpp` 429、447、1546           |
| 三元运算符当控制流                    | `loguru.hpp` 1073 `VLOG_F`                                   |

模板元编程用来把 `ERROR_CONTEXT` 的实参收成指针或按值保存：

```927:940:loguru.hpp
	template <class T>
	struct decay_char_array { using type = T; };

	template <unsigned long long  N>
	struct decay_char_array<const char(&)[N]> { using type = const char*; };
	// ...
	struct make_ec_type { using type = typename make_const_ptr<typename decay_char_array<T>::type>::type; };
```

## 预处理

宏是接口的主体，不是边角。

| 特性                                                         | 位置                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `#if` / `#ifdef` / `#define` / `#undef` / `#error` / `#pragma` | `loguru.hpp` 82–90、139–141                                  |
| `#` 字符串化、`##` 拼接、`__VA_ARGS__`                       | `loguru.hpp` 213–217、1078 `Verbosity_ ## verbosity_name`、1124 |
| `__FILE__`、`__LINE__`、`__func__`、`__COUNTER__`            | `loguru.hpp` 216–219、970、1075、1104                        |
| 宏展开成局部 RAII 对象                                       | `loguru.hpp` 1088 `LOG_SCOPE_F`，969 `ERROR_CONTEXT`         |

## 标准库

| 组件                                      | 位置                                              |
| ----------------------------------------- | ------------------------------------------------- |
| `<chrono>`、`std::thread`                 | `loguru.cpp` 148、214、1489                       |
| `std::atomic`                             | `loguru.cpp` 211、216                             |
| `std::recursive_mutex`、`std::lock_guard` | `loguru.cpp` 200、954                             |
| `std::string`、`std::vector`、`std::pair` | `loguru.cpp` 176–179                              |
| `std::regex`、`std::regex_error`          | `loguru.cpp` 1207–1213                            |
| `std::ostringstream`、`operator<<` 操纵符 | `loguru.hpp` 1265、1270，仅 `LOGURU_WITH_STREAMS` |
| `std::hash<std::thread::id>`              | `loguru.cpp` 1141                                 |

## 编译器扩展

这些不是标准 C++。

| 扩展                                                   | 位置                  |
| ------------------------------------------------------ | --------------------- |
| `__attribute__((format))`、`__attribute__((noreturn))` | `loguru.hpp` 225、239 |
| `__declspec(noreturn)`、SAL 注解 `_In_z_`              | `loguru.hpp` 229、237 |
| `__builtin_expect`                                     | `loguru.hpp` 246–247  |
| `#pragma clang/GCC system_header`                      | `loguru.hpp` 87–90    |
| `__has_feature(cxx_rtti)`、`__GXX_RTTI`、`_CPPRTTI`    | `loguru.hpp` 186–198  |
| `abi::__cxa_demangle`                                  | `loguru.cpp` 1159     |