位置按当前带注释的源码核对。继续在源码里加注释后，行号会再往后移。
 这两个文件以 **C++11** 为上限。`loguru.hpp` 放接口、宏和模板，`loguru.cpp` 放实现。没有 `constexpr`、`static_assert`、`enum class`、`if constexpr`、折叠表达式、结构化绑定，也没有智能指针。

## C++11 语言特性

| 特性                               | English                                                   | 位置                                                         |
| ---------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| `nullptr`                          | nullptr                                                   | `loguru.hpp` 308、701、840；`loguru.cpp` 215–217             |
| `auto`                             | auto                                                      | `loguru.hpp` 352、1049；`loguru.cpp` 190、505、1037          |
| 类型别名 `using`                   | alias declaration                                         | `loguru.hpp` 409 `using Verbosity = int`，1034 `using Printer`；`loguru.cpp` 185–188、1811 |
| 带底层类型的 `enum`                | enumeration with fixed underlying type                    | `loguru.hpp` 423 `enum NamedVerbosity : Verbosity`。这不是 `enum class`，枚举项在 `loguru` 命名空间里 |
| 右值引用、移动构造                 | rvalue reference, move constructor                        | `loguru.hpp` 305 `Text(Text&&)`，824 `LogScopeRAII(LogScopeRAII&&)`，1036 `Printer&&` |
| `= delete`                         | deleted function                                          | `loguru.hpp` 316–318、832–834、999–1002                      |
| `= default`                        | defaulted function                                        | `loguru.hpp` 824 默认移动构造                                |
| 非静态成员默认初始化               | non-static data member initializer                        | `loguru.hpp` 518–546、839–843、1014                          |
| 花括号初始化                       | list initialization                                       | `loguru.hpp` 612 `options = {}`，844 `_name[...] = {}`；`loguru.cpp` 220 `{ 0 }`，1202 `REPLACE_LIST = { ... }` |
| 变参模板                           | variadic template                                         | `loguru.hpp` 383、756、763、853。这些都在 `#if LOGURU_USE_FMTLIB` 里，默认不参与编译 |
| 函数模板显式特化 `template<>`      | explicit specialization                                   | `loguru.hpp` 874–878 `format_value`                          |
| 类模板偏特化、数组引用             | partial class template specialization, reference to array | `loguru.hpp` 1084–1085 `const char(&)[N]`                    |
| `decltype`                         | decltype                                                  | `loguru.hpp` 1146；`loguru.cpp` 168                          |
| `override`                         | override                                                  | `loguru.hpp` 1047                                            |
| `noexcept(false)`                  | noexcept specifier                                        | `loguru.hpp` 1450、1485；定义在 `loguru.cpp` 1777、1783。析构被允许抛异常 |
| `thread_local`                     | thread storage duration                                   | `loguru.cpp` 1085、1817                                      |
| lambda                             | lambda expression                                         | `loguru.cpp` 229 立即调用，1540 作为线程入口                 |
| 范围 `for`                         | range-based for                                           | `loguru.cpp` 1052、1224 `auto&&`、1525、1657、1875           |
| 原始字符串 `R"(...)"`              | raw string literal                                        | `loguru.cpp` 1250、1257                                      |
| 列表初始化 `std::initializer_list` | initializer list                                          | `loguru.cpp` 1202                                            |

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
| 命名空间、匿名命名空间                | namespace, anonymous namespace                         | `loguru.hpp` 215–219、289、294                               |
| `class` / `struct`、访问控制          | class, struct, access control                          | `Text` 297，`Message` 457，`LogScopeRAII` 783，`EcEntryBase` 994 |
| 继承、虚析构、纯虚函数                | inheritance, virtual destructor, pure virtual function | `loguru.hpp` 998、1004、1031 `EcEntryData : public EcEntryBase` |
| `explicit`、构造初始化列表            | explicit constructor, member initializer list          | `loguru.hpp` 300、1037、1449                                 |
| RAII                                  | RAII                                                   | `Text`、`LogScopeRAII`、`EcEntryBase`、`StreamLogger`；`loguru.cpp` 978 `lock_guard` |
| `const` 成员函数                      | const member function                                  | `loguru.hpp` 335、348、1004                                  |
| 函数重载                              | function overloading                                   | `loguru.hpp` 402 与 407 的 `textprintf`；1461 与 1468 的 `operator<<` |
| 运算符重载                            | operator overloading                                   | `loguru.hpp` 1461 `operator<<`，返回 `*this` 形成链式调用    |
| 函数指针 `typedef`                    | pointer to function                                    | `loguru.hpp` 496–501                                         |
| 默认参数                              | default argument                                       | `loguru.hpp` 612、701–702                                    |
| `static` 成员函数、`static` 局部/全局 | static member function, static storage duration        | `loguru.hpp` 548 `SignalOptions::none()`；`loguru.cpp` 209–229 |
| `extern` 全局                         | external linkage                                       | `loguru.hpp` 475 起一组 `g_*`，定义在 `loguru.cpp`           |
| `inline`                              | inline function                                        | `loguru.hpp` 874；`loguru.cpp` 315                           |
| 前向声明                              | forward declaration                                    | `loguru.hpp` 986 `struct StringStream`，定义在 `loguru.cpp` 1798 |
| 引用参数                              | reference parameter                                    | 贯穿 `const Message&`、`const T&`                            |
| `static_cast` / `reinterpret_cast`    | explicit type conversion                               | `loguru.hpp` 1149；`loguru.cpp` 315、359、482、514           |
| `new` / `delete`                      | new-expression, delete-expression                      | `loguru.cpp` 359、711、861、1527、1540、1831                 |
| `try` / `catch`，含 `catch (...)`     | exception handling                                     | `loguru.cpp` 510、1249、2078                                 |
| `typeid`（RTTI）                      | runtime type identification                            | `loguru.cpp` 1197；头文件 190–211 用编译器宏决定是否开启     |
| C 风格变参 `...`、`va_list`           | variadic function, `va_list`                           | `loguru.hpp` 402、771、775；`loguru.cpp` 453、472、1598      |
| 三元运算符当控制流                    | conditional operator                                   | `loguru.hpp` 1263 `VLOG_F`                                   |

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
| `#if` / `#ifdef` / `#define` / `#undef` / `#error` / `#pragma` | preprocessing directives                   | `loguru.hpp` 83–94、140                                      |
| `#` 字符串化、`##` 拼接、`__VA_ARGS__`                       | stringizing, token pasting, variadic macro | `loguru.hpp` 225–226、275、1268 `Verbosity_ ## verbosity_name` |
| `__FILE__`、`__LINE__`、`__func__`、`__COUNTER__`            | predefined macro                           | `loguru.hpp` 228、1148、1265、1294                           |
| 宏展开成局部 RAII 对象                                       | macro expanding to a scoped RAII object    | `loguru.hpp` 1291 `LOG_SCOPE_F`，1145 `ERROR_CONTEXT`        |

## 标准库

| 组件                                      | English                                | 位置                                              |
| ----------------------------------------- | -------------------------------------- | ------------------------------------------------- |
| `<chrono>`、`std::thread`                 | chrono library, thread support library | `loguru.cpp` 157、223、1540                       |
| `std::atomic`                             | atomic operations library              | `loguru.cpp` 220、225                             |
| `std::recursive_mutex`、`std::lock_guard` | mutual exclusion, RAII lock            | `loguru.cpp` 209、978                             |
| `std::string`、`std::vector`、`std::pair` | string, sequence container, pair       | `loguru.cpp` 185–188                              |
| `std::regex`、`std::regex_error`          | regular expressions library            | `loguru.cpp` 1250–1259                            |
| `std::ostringstream`、`operator<<` 操纵符 | string stream, stream manipulator      | `loguru.hpp` 1461、1478，仅 `LOGURU_WITH_STREAMS` |
| `std::hash<std::thread::id>`              | hash function object                   | `loguru.cpp` 1170                                 |

## 编译器扩展

这些不是标准 C++。

| 扩展                                                   | English                            | 位置                  |
| ------------------------------------------------------ | ---------------------------------- | --------------------- |
| `__attribute__((format))`、`__attribute__((noreturn))` | GNU function attribute             | `loguru.hpp` 239、262 |
| `__declspec(noreturn)`、SAL 注解 `_In_z_`              | Microsoft declspec, SAL annotation | `loguru.hpp` 252、260 |
| `__builtin_expect`                                     | GCC branch prediction builtin      | `loguru.hpp` 269–270  |
| `#pragma clang/GCC system_header`                      | system header pragma               | `loguru.hpp` 88、94   |
| `__has_feature(cxx_rtti)`、`__GXX_RTTI`、`_CPPRTTI`    | RTTI feature-test macro            | `loguru.hpp` 199–211  |
| `abi::__cxa_demangle`                                  | Itanium ABI name demangling        | `loguru.cpp` 1188     |
