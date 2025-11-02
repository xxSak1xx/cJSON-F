# Notes on CJson

## 项目架构

| 类别         | 主要目录/文件                                                     | 作用          |
| ---------- | ----------------------------------------------------------- | ----------- |
| **源码文件**   | `cJSON.c`, `cJSON.h`, `cJSON_Utils.c`, `cJSON_Utils.h`      | 核心实现与工具函数   |
| **测试与示例**  | `test.c`, `tests/`                                          | 测试程序与单元测试   |
| **构建系统**   | `CMakeLists.txt`, `Makefile`                                | 提供多平台编译方式   |
| **持续集成配置** | `.github/workflows/`, `.travis.yml`, `appveyor.yml`         | CI/CD 自动测试  |
| **项目配置**   | `.editorconfig`, `.gitignore`, `.gitattributes`, `.vscode/` | 编辑器与 Git 配置 |
| **文档与说明**  | `README.md`, `CONTRIBUTING.md`, `CHANGES.md`, `LICENSE`     | 项目说明与开发规范   |
| **调试辅助**   | `valgrind.supp`, `library_config/`, `fuzzing/`              | 内存检查与模糊测试支持 |

其中CI/CD表示：
- Continuous Integration：开发者频繁地（例如每次提交）将代码合并到主分支，并通过自动化构建与测试来验证代码质量，如构建项目、单元测试、静态测试、模糊测试等。若有步骤发生错误，则阻止提交。
- Continuous Delivery：代码通过测试后自动打包、准备好随时发布（例如上传到测试服务器或生成安装包）。需要人工确认上线。
- Continuous Deployment：测试通过后自动部署到生产环境，无需人工确认。

yaml(Yaml Ain't markup Language)是一种数据序列化语言（data serialization language），
主要用于配置文件、数据传输和自动化工具定义，如CI/CD。

Fuzzing则表示模糊测试，通过向程序输入大量随机或半随机的数据（称为fuzz数据或模糊输入），观察程序是否出现崩溃、内存错误、逻辑异常、安全漏洞等。

## cJSON.h

```C
#ifdef __cplusplus
extern "C"
{ ... }
```
这是在C/C++混合编译环境下非常常见的写法。
若存在C++编译器默认定义的宏，则按照C的方式导出符号，不要进行名字修饰[^1]，以实现与C的兼容。

[^1]:名字修饰（Name Mangling） 是编译器为支持 C++ 特性（如重载、命名空间、类成员）而对符号名进行编码的机制。

```C
#define CJSON_CDECL __cdecl
#define CJSON_STDCALL __stdcall
```

> 其中"cdecl"表示**C Declaration**。

Windows函数调用约定：
|__cdecl|__stdcall|
|:---:|:---:|
|C 默认调用约定（参数由调用者清理栈）|Windows API 常用调用约定（参数由被调函数清理栈）|

```C
#if !defined(CJSON_HIDE_SYMBOLS) && !defined(CJSON_IMPORT_SYMBOLS) && !defined(CJSON_EXPORT_SYMBOLS)
#define CJSON_EXPORT_SYMBOLS
#endif
#if defined(CJSON_HIDE_SYMBOLS)
#define CJSON_PUBLIC(type)   type CJSON_STDCALL
#elif defined(CJSON_EXPORT_SYMBOLS)
#define CJSON_PUBLIC(type)   __declspec(dllexport) type CJSON_STDCALL
#elif defined(CJSON_IMPORT_SYMBOLS)
#define CJSON_PUBLIC(type)   __declspec(dllimport) type CJSON_STDCALL
#endif
```
> "declspec"表示 Declare Specifier，即声明说明符。这是 Microsoft Visual C++ (MSVC) 编译器特有的扩展关键字，用来给变量、函数或类型加上额外的声明属性。

这几行代码处理Windows环境下符号的可见性。
- HIDE_SYMBOLS表示隐藏符号，只能库内部使用
- IMPORT_SYMBOLS表示导入外部的函数符号，在链接阶段从dll中查找
- EXPORT_SYMBOLS表示导出函数符号，将该函数加入dll的导出表中，使其他程序可以找到

```C
/* !__WINDOWS__ */
#define CJSON_CDECL
#define CJSON_STDCALL

#if (defined(__GNUC__) || defined(__SUNPRO_CC) || defined (__SUNPRO_C)) && defined(CJSON_API_VISIBILITY)
#define CJSON_PUBLIC(type)   __attribute__((visibility("default"))) type
#else
#define CJSON_PUBLIC(type) type
#endif
```
这段代码用于处理非Windows系统时函数的导出。
如果有GCC或SunStudio编译器，且用户启用了控制符号可见性的宏，则表示导出函数（默认可见），等效于__declspec(dllexport)

```C
#define cJSON_Invalid (0)
#define cJSON_False  (1 << 0)
#define cJSON_True   (1 << 1)
#define cJSON_NULL   (1 << 2)
#define cJSON_Number (1 << 3)
#define cJSON_String (1 << 4)
#define cJSON_Array  (1 << 5)
#define cJSON_Object (1 << 6)
#define cJSON_Raw    (1 << 7) /* raw json */

#define cJSON_IsReference 256
#define cJSON_StringIsConst 512
```
此为cJSON类别的**掩码**[^2]。前八种是类型，后两种为附加属性。

[^2]:用每一位（bit）表示一个布尔状态。

| 优点       | 解释                                      |                                |
| -------- | --------------------------------------- | ------------------------------ |
| **节省内存** | 一个 `int` 就能存 32 个布尔状态（64 位系统甚至能存 64 个）。 |                                |
| **操作快**  | 位运算非常高效（CPU 级别指令），不需要循环或数组。             |                                |
| **易于组合** | 通过按位与、按位或操作可以轻松设置或检测多个标志。 |
| **易于扩展** | 想加新标志位，只要定义新的 `1 << n`。不用改结构体或函数接口。     |                                |


```C
typedef struct cJSON_Hooks
{
      /* malloc/free are CDECL on Windows regardless of the default calling convention of the compiler, so ensure the hooks allow passing those functions directly. */
      void *(CJSON_CDECL *malloc_fn)(size_t sz);
      void (CJSON_CDECL *free_fn)(void *ptr);
} cJSON_Hooks;
```
此为支持用户自定义内存分配函数

```C
CJSON_PUBLIC(cJSON *) cJSON_GetObjectItem(const cJSON * const object, const char * const string);
```
此处参数`const cJSON * const object`, 前一个为修饰对象内容不可更改，后一个为修饰指针指向不可更改。

## cJSON.c

```C
#if defined(_MSC_VER)
#pragma warning (pop)
#endif
#ifdef __GNUC__
#pragma GCC visibility pop
#endif
```

这段这一段代码的目的是恢复编译器的原始状态。
MSVC：恢复警告设置，GCC：恢复符号可见性。这样库的内部配置不会影响用户项目，保证跨平台兼容性。

```C
#ifndef isinf
#define isinf(d) (isnan((d - d)) && !isnan(d))
#endif
#ifndef isnan
#define isnan(d) (d != d)
#endif

#ifndef NAN
#ifdef _WIN32
#define NAN sqrt(-1.0)
#else
#define NAN 0.0/0.0
#endif
```
前一段用于判断浮点数是否为`NaN`（Not
 a Number）或无穷。[^3]
 
 [^3]:在 IEEE 754 浮点标准中，NaN 与自身不相等。所以 d != d 如果为真，说明 d 是 NaN; 而d为无穷大时，d - d = NaN

 ```C
 typedef struct internal_hooks
{
    void *(CJSON_CDECL *allocate)(size_t size);
    void (CJSON_CDECL *deallocate)(void *pointer);
    void *(CJSON_CDECL *reallocate)(void *pointer, size_t size);
} internal_hooks;
 ```
 内部`hook`函数集合，用于注册用户自定义的内存管理函数。
 **`realloc`函数并没有暴露给用户，只有在`free`和`malloc`函数均为标准库中的情况下才启用。这是防止自定义与系统内存管理函数混用导致不能正确识别并重新分配内存。**
 
 ```C
 static unsigned char get_decimal_point(void)
{
#ifdef ENABLE_LOCALES
    struct lconv *lconv = localeconv();
    return (unsigned char) lconv->decimal_point[0];
#else
    return '.';
#endif
}
```
这段代码使我再次感概此库的兼容性之强，不愧为GitHub上经典的C项目。这一内部函数是用来获取小数点的，利用条件编译确定了当前系统的正确形式[^4]。

[^4]:德、法、意、俄等地区，其小数点表示为","

如果`ENABLE_LOCALES`启用的话，就从`lconv` (local conventions)中调用当地约定的小数点，否则就采用JSON标准中规定的"."为小数点。

```C
typedef struct
{
    const unsigned char *content;
    size_t length;
    size_t offset;
    size_t depth; /* How deeply nested (in arrays/objects) is the input at the current offset. */
    internal_hooks hooks;
} parse_buffer;
/* check if the given size is left to read in a given parse buffer (starting with 1) */
#define can_read(buffer, size) ((buffer != NULL) && (((buffer)->offset + size) <= (buffer)->length))
/* check if the buffer can be accessed at the given index (starting with 0) */
#define can_access_at_index(buffer, index) ((buffer != NULL) && (((buffer)->offset + index) < (buffer)->length))
#define cannot_access_at_index(buffer, index) (!can_access_at_index(buffer, index))
/* get a pointer to the buffer at the position */
#define buffer_at_offset(buffer) ((buffer)->content + (buffer)->offset)
```
这种缓冲区的设计十分精妙且常见。使用全局变量保存算法数据不仅杂乱，还会在多个文件共用一个库的时候产生竞争，因此就要设计`buffer`来统一管理解析。下面是与该`buffer`相关的一些辅助宏。

```C
goto fail;
···
fail:
···
```

此库中多处用到这种写法，但跳转层数少、逻辑清晰、程序出口统一，是可以接受的处理。

```C
#if defined(__clang__) || (defined(__GNUC__)  && ((__GNUC__ > 4) || ((__GNUC__ == 4) && (__GNUC_MINOR__ > 5))))
    #pragma GCC diagnostic push
#endif
#ifdef __GNUC__
#pragma GCC diagnostic ignored "-Wcast-qual"
#endif
/* helper function to cast away const */
static void* cast_away_const(const void* string)
{
    return (void*)string;
}
#if defined(__clang__) || (defined(__GNUC__)  && ((__GNUC__ > 4) || ((__GNUC__ == 4) && (__GNUC_MINOR__ > 5))))
    #pragma GCC diagnostic pop
#endif
```

这段代码用于临时忽略编译器`const` 警告，用于衔接旧库函数或API与现代C风格。

```C
static void skip_oneline_comment(char **input)
{
    *input += static_strlen("//");

    for (; (*input)[0] != '\0'; ++(*input))
    {
        if ((*input)[0] == '\n') {
            *input += static_strlen("\n");
            return;
        }
    }
}

static void skip_multiline_comment(char **input)
{
    *input += static_strlen("/*");

    for (; (*input)[0] != '\0'; ++(*input))
    {
        if (((*input)[0] == '*') && ((*input)[1] == '/'))
        {
            *input += static_strlen("*/");
            return;
        }
    }
}
```

此处比较人性化地添加了注释选项，尽管json的官方标准并不支持注释。

`inline`和`restrict`关键字：
`inline`修饰的函数通常是短小、常用到的。这一关键字会**建议**编译器将函数在调用时直接展开，而不是生成真正的函数调用指令。这可以消除函数调用的栈开销，加快执行速度。
`restrict`则是对指针的限定，用于通知编译器该指针访问的对象不会被其他指针访问或修改，使之可以激进优化内存读写和寄存器缓存。
**对RISC-V而言，可以使内存访问次数明显降低，五级流水线高速运转。**

> 此处，我深深地感受到**计算机组成**理论与实践的联系，尤其涉及到刚刚学习到的流水线相关的知识。现在，我可以从更底层的角度去理解C乃至其他语言，甚至能从底层的视角去看待高层的应用。我为此感到由衷的高兴。

