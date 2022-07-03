# 从零开始的 JSON 库教程
## 一　启程
### 1　JSON 是什么
[JSON](https://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf)（JavaScript Object Notation）是用于数据交换的文本格式。
JSON 树状结构且只包含 6 种数据类型：
- null
- boolean：true/false
- number：一般的浮点数
- string："..."
- array：[...]
- object：{...}

JSON 库有 3 个需求：
1. 把 JSON 文本解析成树状结构（parse）。
2. 提供接口访问数据（access）。
3. 把数据结构转换回 JSON 文本（stringify）。

### 2　搭建编译环境
头文件（header file）含有对外的类型与 API 声明。
实现文件（implementation file）含有内部类型声明和函数实现，会编译成库。
一般在 CMake 工程内新建 build 文件夹保存生成文件，-DCMAKE_BUILD_TYPE=Debug 标志可生成 Debug 工程。

### 3　头文件与 API 设计
C 语言使用 `#include` 引入头文件的类型与函数声明。
头文件使用宏进行 include 防范（include guard）。宏名唯一，一般以 `_H__` 作为后缀，命名为 `项目名_目录_文件名_H__` 的形式。
枚举类型（enumeration type）一般使用项目简写作为标识符前缀，通常全大写，而类型及函数通常全小写。这里声明 `lept_type` 类型表示 JSON 类型。
```c
typedef enum { LEPT_NULL, LEPT_FALSE, LEPT_TRUE, LEPT_NUMBER, LEPT_STRING, LEPT_ARRAY, LEPT_OBJECT } lept_type;
```
JSON 的数据实现为数，节点用 `lept_value` 表示。`typedef` 简化定义。
```c
typedef struct {
  lept_type type;
} lept_value;
```
解析 API：
```c
int lept_parse(lept_value* v, char const* json);
```
JSON 文本使用 C 字符串（空结尾字符串/null-terminated string），使用 `char const*` 类型防止改变。
返回值为下列枚举，无错返回 `LEPT_PARSE_OK`。
```c
enum {
  LEPT_PARSE_OK = 0,
  LEPT_PARSE_EXPECT_VALUE,
  LEPT_PARSE_INVALID_VALUE,
  LEPT_PARSE_ROOT_NOT_SINGULAR
};
```
访问函数：
```c
lept_type lept_get_type(lept_value const* v);
```
### 4　JSON 语法子集
```ABNF
JSON-text = ws value ws
ws = *(%x20 / %x09 / %x0A / %x0D)
value = null / false / true
null  = "null"
false = "false"
true  = "true"
```
`%xhh` 表示 16 进制表示的字符，`/` 表示多选一，`*` 表示零或多个，`()` 用于分组。
JSON 文本由 3 部分组成，空白（whitespace）+ 值 + 空白。
空白是零或多个空格（space U+0020）、制表符（tab U+0009）、换行符（LF U+000A）、回车符（CR U+000D）构成的。
值包括 null、false 或 true，它们有对应的字面量（literal）。
当出现 JSON 语法错误，我们将会返回错误码：
- `LEPT_PARSE_EXPECT_VALUE`：只含有空白。
- `LEPT_PARSE_ROOT_NOT_SINGULAR`：在值后面的空白后还有字符。
- `LEPT_PARSE_INVALID_VALUE`：值不是三种字面值。

### 5　单元测试
单元测试（unit testing）等测试为自动测试，可确保其他人修改代码后，源代码依旧正确（回归测试/regression testing）。
测试驱动开发（test-driven development，TDD）：
1. 加入一个测试。
2. 运行所有测试，新测试应该失败。
3. 编写实现代码。
4. 运行测试，若失败返回 3。
5. 重构代码。
6. 回到 1。

### 6　宏的编写技巧
反斜杠代表该行未结束，串接下一行。
宏内语句（statement）多于一个时，使用 `do { ... } while(0)` 包裹为单个语句。

### 7　实现解析器
使用 `lept_context` 结构体传递参数：
```c
typedef struct {
  char const* json;
} lept_context;

/* .. */

/* JSON-text = ws value ws */
int lept_parse(lept_value* v, char const* json) {
  lept_context c;
  assert(v != NULL);
  c.json = json;
  v->type = LEPT_NULL;
  lept_parse_whitespace(&c);
  return lept_parse_value(&c, v);
}
```
若 `lept_parse()` 失败，则会置 `v` 为 `null` 类型。
leptjson 是一个手写递归下降解析器（recursive descent parser）。因为 JSON 语法特别简单，所以无需分词器（tokenizer），只需检测字符。
- n -> null
- t -> true
- f -> false
- " -> string
- 0-9/- -> number
- [ -> array
- { -> object

```c
#define EXPECT(c, ch) do { assert(*c->json == (ch)); c->json++; } while(0)

/* ws = *(%x20 / %x09 / %x0A / %x0D) */
static void lept_parse_whitespace(lept_context* c) {
    const char *p = c->json;
    while (*p == ' ' || *p == '\t' || *p == '\n' || *p == '\r')
        p++;
    c->json = p;
}

/* null  = "null" */
static int lept_parse_null(lept_context* c, lept_value* v) {
    EXPECT(c, 'n');
    if (c->json[0] != 'u' || c->json[1] != 'l' || c->json[2] != 'l')
        return LEPT_PARSE_INVALID_VALUE;
    c->json += 3;
    v->type = LEPT_NULL;
    return LEPT_PARSE_OK;
}

/* value = null / false / true */
static int lept_parse_value(lept_context* c, lept_value* v) {
    switch (*c->json) {
        case 'n':  return lept_parse_null(c, v);
        case '\0': return LEPT_PARSE_EXPECT_VALUE;
        default:   return LEPT_PARSE_INVALID_VALUE;
    }
}
```

### 8　关于断言
断言（assertion）是 C 语言常用的防御式编程方式，减少错误。在函数开始检测参数，在函数后检测上下文是否正确。
C 语言有 `assert()` 宏（`assert.h`）进行断言。release（`NDEBUG`）不会检测；debug（不定义 `NDEBUG`）会检测条件是否为真，不为真则崩溃。
含副作用的代码不可断言，影响 release 与 debug 行为。

### 9　总结与练习

### 10　常见问答
- 命名项目时，google 看名称是否独特。
- `__LINE__` 宏表示编译时的行号。