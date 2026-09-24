位置按当前带注释的源码核对。继续在源码里加注释后，行号会再往后移。
 这两个文件以 **C++11** 为上限。`loguru.hpp` 放接口、宏和模板，`loguru.cpp` 放实现。没有 `constexpr`、`static_assert`、`enum class`、`if constexpr`、折叠表达式、结构化绑定，也没有智能指针。

## C++11 语言特性

| 特性                               | English                                                   | 位置                                                         |
| ---------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| `nullptr`                          | nullptr                                                   | [loguru.hpp:308](loguru.hpp#L308)、[701](loguru.hpp#L701)、[840](loguru.hpp#L840)；[loguru.cpp:215](loguru.cpp#L215)–[217](loguru.cpp#L217) |
| `auto`                             | auto                                                      | [loguru.hpp:352](loguru.hpp#L352)、[1049](loguru.hpp#L1049)；[loguru.cpp:190](loguru.cpp#L190)、[505](loguru.cpp#L505)、[1037](loguru.cpp#L1037) |
| 类型别名 `using`                   | alias declaration                                         | [loguru.hpp:409](loguru.hpp#L409) `using Verbosity = int`，[1034](loguru.hpp#L1034) `using Printer`；[loguru.cpp:185](loguru.cpp#L185)–[188](loguru.cpp#L188)、[1811](loguru.cpp#L1811) |
| 带底层类型的 `enum`                | enumeration with fixed underlying type                    | [loguru.hpp:423](loguru.hpp#L423) `enum NamedVerbosity : Verbosity`。这不是 `enum class`，枚举项在 `loguru` 命名空间里 |
| 右值引用、移动构造                 | rvalue reference, move constructor                        | [loguru.hpp:305](loguru.hpp#L305) `Text(Text&&)`，[824](loguru.hpp#L824) `LogScopeRAII(LogScopeRAII&&)`，[1036](loguru.hpp#L1036) `Printer&&` |
| `= delete`                         | deleted function                                          | [loguru.hpp:316](loguru.hpp#L316)–[318](loguru.hpp#L318)、[832](loguru.hpp#L832)–[834](loguru.hpp#L834)、[999](loguru.hpp#L999)–[1002](loguru.hpp#L1002) |
| `= default`                        | defaulted function                                        | [loguru.hpp:824](loguru.hpp#L824) 默认移动构造               |
| 非静态成员默认初始化               | non-static data member initializer                        | [loguru.hpp:518](loguru.hpp#L518)–[546](loguru.hpp#L546)、[839](loguru.hpp#L839)–[843](loguru.hpp#L843)、[1014](loguru.hpp#L1014) |
| 花括号初始化                       | list initialization                                       | [loguru.hpp:612](loguru.hpp#L612) `options = {}`，[844](loguru.hpp#L844) `_name[...] = {}`；[loguru.cpp:220](loguru.cpp#L220) `{ 0 }`，[1202](loguru.cpp#L1202) `REPLACE_LIST = { ... }` |
| 变参模板                           | variadic template                                         | [loguru.hpp:383](loguru.hpp#L383)、[756](loguru.hpp#L756)、[763](loguru.hpp#L763)、[853](loguru.hpp#L853)。这些都在 `#if LOGURU_USE_FMTLIB` 里，默认不参与编译 |
| 函数模板显式特化 `template<>`      | explicit specialization                                   | [loguru.hpp:874](loguru.hpp#L874)–[878](loguru.hpp#L878) `format_value` |
| 类模板偏特化、数组引用             | partial class template specialization, reference to array | [loguru.hpp:1084](loguru.hpp#L1084)–[1085](loguru.hpp#L1085) `const char(&)[N]` |
| `decltype`                         | decltype                                                  | [loguru.hpp:1146](loguru.hpp#L1146)；[loguru.cpp:168](loguru.cpp#L168) |
| `override`                         | override                                                  | [loguru.hpp:1047](loguru.hpp#L1047)                          |
| `noexcept(false)`                  | noexcept specifier                                        | [loguru.hpp:1450](loguru.hpp#L1450)、[1485](loguru.hpp#L1485)；定义在 [loguru.cpp:1777](loguru.cpp#L1777)、[1783](loguru.cpp#L1783)。析构被允许抛异常 |
| `thread_local`                     | thread storage duration                                   | [loguru.cpp:1085](loguru.cpp#L1085)、[1817](loguru.cpp#L1817) |
| lambda                             | lambda expression                                         | [loguru.cpp:229](loguru.cpp#L229) 立即调用，[1540](loguru.cpp#L1540) 作为线程入口 |
| 范围 `for`                         | range-based for                                           | [loguru.cpp:1052](loguru.cpp#L1052)、[1224](loguru.cpp#L1224) `auto&&`、[1525](loguru.cpp#L1525)、[1657](loguru.cpp#L1657)、[1875](loguru.cpp#L1875) |
| 原始字符串 `R"(...)"`              | raw string literal                                        | [loguru.cpp:1250](loguru.cpp#L1250)、[1257](loguru.cpp#L1257) |
| 列表初始化 `std::initializer_list` | initializer list                                          | [loguru.cpp:1202](loguru.cpp#L1202)                          |

移动和删除特殊成员集中在 `Text`：

```305:318:loguru.hpp
		Text(Text&& t)   //- 定义移动构造函数
		{
			_str = t._str;
			t._str = nullptr;
			//^ nullptr 是 C++11 的空指针常量，类型是 std::nullptr_t。
			//^ 它可以隐式转换成任何对象指针、函数指针或成员指针，转换结果是对应类型的空指针。
			//^ s_fatal_handler = nullptr 表示这个函数指针当前不指向任何函数。
			//^ 用指针之前要判断它是不是 nullptr，对空指针解引用是未定义行为。
			//^ 它不是整数。NULL 常被定义成 0，重载时可能匹配到 int 参数；nullptr 只会匹配指针参数。
			//^ 和指针比较、赋值都可以：p == nullptr、p = nullptr。
		}
		Text(Text& t) = delete; //- 禁止拷贝构造函数
		Text& operator=(Text& t) = delete; //- 禁止拷贝赋值运算符
		void operator=(Text&& t) = delete; //- 禁止移动赋值运算符
```

## C++98 仍在大量使用的机制

| 特性                                  | English                                                | 位置                                                         |
| ------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| 命名空间、匿名命名空间                | namespace, anonymous namespace                         | [loguru.hpp:215](loguru.hpp#L215)–[219](loguru.hpp#L219)、[289](loguru.hpp#L289)、[294](loguru.hpp#L294) |
| `class` / `struct`、访问控制          | class, struct, access control                          | [Text](loguru.hpp#L297)，[Message](loguru.hpp#L457)，[LogScopeRAII](loguru.hpp#L783)，[EcEntryBase](loguru.hpp#L994) |
| 继承、虚析构、纯虚函数                | inheritance, virtual destructor, pure virtual function | [loguru.hpp:998](loguru.hpp#L998)、[1004](loguru.hpp#L1004)、[1031](loguru.hpp#L1031) `EcEntryData : public EcEntryBase` |
| `explicit`、构造初始化列表            | explicit constructor, member initializer list          | [loguru.hpp:300](loguru.hpp#L300)、[1037](loguru.hpp#L1037)、[1449](loguru.hpp#L1449) |
| RAII                                  | RAII                                                   | [Text](loguru.hpp#L297)、[LogScopeRAII](loguru.hpp#L783)、[EcEntryBase](loguru.hpp#L994)、[StreamLogger](loguru.hpp#L1446)；[loguru.cpp:978](loguru.cpp#L978) `lock_guard` |
| `const` 成员函数                      | const member function                                  | [loguru.hpp:335](loguru.hpp#L335)、[348](loguru.hpp#L348)、[1004](loguru.hpp#L1004) |
| 函数重载                              | function overloading                                   | [loguru.hpp:402](loguru.hpp#L402) 与 [407](loguru.hpp#L407) 的 `textprintf`；[1461](loguru.hpp#L1461) 与 [1468](loguru.hpp#L1468) 的 `operator<<` |
| 运算符重载                            | operator overloading                                   | [loguru.hpp:1461](loguru.hpp#L1461) `operator<<`，返回 `*this` 形成链式调用 |
| 函数指针 `typedef`                    | pointer to function                                    | [loguru.hpp:496](loguru.hpp#L496)–[501](loguru.hpp#L501)     |
| 默认参数                              | default argument                                       | [loguru.hpp:612](loguru.hpp#L612)、[701](loguru.hpp#L701)–[702](loguru.hpp#L702) |
| `static` 成员函数、`static` 局部/全局 | static member function, static storage duration        | [loguru.hpp:548](loguru.hpp#L548) `SignalOptions::none()`；[loguru.cpp:209](loguru.cpp#L209)–[229](loguru.cpp#L229) |
| `extern` 全局                         | external linkage                                       | [loguru.hpp:475](loguru.hpp#L475) 起一组 `g_*`，定义在 [loguru.cpp:192](loguru.cpp#L192) |
| `inline`                              | inline function                                        | [loguru.hpp:874](loguru.hpp#L874)；[loguru.cpp:315](loguru.cpp#L315) |
| 前向声明                              | forward declaration                                    | [loguru.hpp:986](loguru.hpp#L986) `struct StringStream`，定义在 [loguru.cpp:1798](loguru.cpp#L1798) |
| 引用参数                              | reference parameter                                    | 贯穿 [const Message&](loguru.hpp#L496)、`const T&`           |
| `static_cast` / `reinterpret_cast`    | explicit type conversion                               | [loguru.hpp:1149](loguru.hpp#L1149)；[loguru.cpp:315](loguru.cpp#L315)、[359](loguru.cpp#L359)、[482](loguru.cpp#L482)、[514](loguru.cpp#L514) |
| `new` / `delete`                      | new-expression, delete-expression                      | [loguru.cpp:359](loguru.cpp#L359)、[711](loguru.cpp#L711)、[861](loguru.cpp#L861)、[1527](loguru.cpp#L1527)、[1540](loguru.cpp#L1540)、[1831](loguru.cpp#L1831) |
| `try` / `catch`，含 `catch (...)`     | exception handling                                     | [loguru.cpp:510](loguru.cpp#L510)、[1249](loguru.cpp#L1249)、[2078](loguru.cpp#L2078) |
| `typeid`（RTTI）                      | runtime type identification                            | [loguru.cpp:1197](loguru.cpp#L1197)；头文件 [190](loguru.hpp#L190)–[211](loguru.hpp#L211) 用编译器宏决定是否开启 |
| C 风格变参 `...`、`va_list`           | variadic function, `va_list`                           | [loguru.hpp:402](loguru.hpp#L402)、[771](loguru.hpp#L771)、[775](loguru.hpp#L775)；[loguru.cpp:453](loguru.cpp#L453)、[472](loguru.cpp#L472)、[1598](loguru.cpp#L1598) |
| 三元运算符当控制流                    | conditional operator                                   | [loguru.hpp:1263](loguru.hpp#L1263) `VLOG_F`                 |

模板元编程用来把 `ERROR_CONTEXT` 的实参收成指针或按值保存：

```1081:1096:loguru.hpp
	template <class T>
	struct decay_char_array { using type = T; };

	template <unsigned long long  N>
	struct decay_char_array<const char(&)[N]> { using type = const char*; };
	// ...
	struct make_ec_type { using type = typename make_const_ptr<typename decay_char_array<T>::type>::type; };
```

## 预处理

宏是接口的主体，不是边角。

| 特性                                                         | English                                    | 位置                                                         |
| ------------------------------------------------------------ | ------------------------------------------ | ------------------------------------------------------------ |
| `#if` / `#ifdef` / `#define` / `#undef` / `#error` / `#pragma` | preprocessing directives                   | [loguru.hpp:83](loguru.hpp#L83)–[94](loguru.hpp#L94)、[140](loguru.hpp#L140) |
| `#` 字符串化、`##` 拼接、`__VA_ARGS__`                       | stringizing, token pasting, variadic macro | [loguru.hpp:225](loguru.hpp#L225)–[226](loguru.hpp#L226)、[275](loguru.hpp#L275)、[1268](loguru.hpp#L1268) `Verbosity_ ## verbosity_name` |
| `__FILE__`、`__LINE__`、`__func__`、`__COUNTER__`            | predefined macro                           | [loguru.hpp:228](loguru.hpp#L228)、[1148](loguru.hpp#L1148)、[1265](loguru.hpp#L1265)、[1294](loguru.hpp#L1294) |
| 宏展开成局部 RAII 对象                                       | macro expanding to a scoped RAII object    | [loguru.hpp:1291](loguru.hpp#L1291) `LOG_SCOPE_F`，[1145](loguru.hpp#L1145) `ERROR_CONTEXT` |

## 标准库

| 组件                                      | English                                | 位置                                              |
| ----------------------------------------- | -------------------------------------- | ------------------------------------------------- |
| `<chrono>`、`std::thread`                 | chrono library, thread support library | [loguru.cpp:157](loguru.cpp#L157)、[223](loguru.cpp#L223)、[1540](loguru.cpp#L1540) |
| `std::atomic`                             | atomic operations library              | [loguru.cpp:220](loguru.cpp#L220)、[225](loguru.cpp#L225) |
| `std::recursive_mutex`、`std::lock_guard` | mutual exclusion, RAII lock            | [loguru.cpp:209](loguru.cpp#L209)、[978](loguru.cpp#L978) |
| `std::string`、`std::vector`、`std::pair` | string, sequence container, pair       | [loguru.cpp:185](loguru.cpp#L185)–[188](loguru.cpp#L188) |
| `std::regex`、`std::regex_error`          | regular expressions library            | [loguru.cpp:1250](loguru.cpp#L1250)–[1259](loguru.cpp#L1259) |
| `std::ostringstream`、`operator<<` 操纵符 | string stream, stream manipulator      | [loguru.hpp:1461](loguru.hpp#L1461)、[1478](loguru.hpp#L1478)，仅 `LOGURU_WITH_STREAMS` |
| `std::hash<std::thread::id>`              | hash function object                   | [loguru.cpp:1170](loguru.cpp#L1170)               |

## 编译器扩展

这些不是标准 C++。

| 扩展                                                   | English                            | 位置                  |
| ------------------------------------------------------ | ---------------------------------- | --------------------- |
| `__attribute__((format))`、`__attribute__((noreturn))` | GNU function attribute             | [loguru.hpp:239](loguru.hpp#L239)、[262](loguru.hpp#L262) |
| `__declspec(noreturn)`、SAL 注解 `_In_z_`              | Microsoft declspec, SAL annotation | [loguru.hpp:252](loguru.hpp#L252)、[260](loguru.hpp#L260) |
| `__builtin_expect`                                     | GCC branch prediction builtin      | [loguru.hpp:269](loguru.hpp#L269)–[270](loguru.hpp#L270) |
| `#pragma clang/GCC system_header`                      | system header pragma               | [loguru.hpp:88](loguru.hpp#L88)、[94](loguru.hpp#L94) |
| `__has_feature(cxx_rtti)`、`__GXX_RTTI`、`_CPPRTTI`    | RTTI feature-test macro            | [loguru.hpp:199](loguru.hpp#L199)–[211](loguru.hpp#L211) |
| `abi::__cxa_demangle`                                  | Itanium ABI name demangling        | [loguru.cpp:1188](loguru.cpp#L1188) |
